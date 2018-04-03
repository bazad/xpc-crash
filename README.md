xpc-crash
===================================================================================================

<!-- Brandon Azad -->

xpc-crash is a proof-of-concept exploit for an out-of-bounds read in libxpc that can be used to
force any XPC service to crash on iOS versions prior to 11.3 and macOS versions prior to 10.13.4.


The vulnerability
---------------------------------------------------------------------------------------------------

Prior to macOS 10.13.4, the function `_xpc_dictionary_apply_wire_f`, which is used by
`_xpc_dictionary_deserialize` to deserialize an XPC dictionary from the wire, contained the
following code to iteratively deserialize each entry of the dictionary:
```c
void _xpc_dictionary_apply_wire_f
(
        OS_xpc_dictionary *xdict,
        OS_xpc_serializer *xserializer,
        const void *context,
        bool (*applier_fn)(const char *, OS_xpc_serializer *, const void *)
)
{
...
    uint64_t count = (unsigned int)*serialized_dict_count;
    if ( count )
    {
        uint64_t depth = xserializer->depth;
        uint64_t index = 0;
        do
        {
            const char *key = _xpc_serializer_read(xserializer, 0, 0, 0);
            size_t keylen = strlen(key);
            _xpc_serializer_advance(xserializer, keylen + 1);
            if ( !applier_fn(key, xserializer, context) )
                break;
            xserializer->depth = depth;
            ++index;
        }
        while ( index < count );
    }
...
}
```

The problem is that the use of an unchecked `strlen` on attacker-controlled data allows the key for
the serialized dictionary entry to extend beyond the end of the data buffer. This means the XPC
service deserializing the dictionary will crash, either when `strlen` dereferences out-of-bounds
memory or when `_xpc_serializer_advance` tries to advance the serializer past the end of the
supplied data.


Usage
---------------------------------------------------------------------------------------------------

To build, run `make`. See the top of the Makefile for various build options.

Run the exploit by specifying the name of the XPC service endpoint to crash on the command line:

	$ ./xpc-crash com.apple.iokit.powerdxpc

You will notice that the process corresponding to the XPC service, which in this example is powerd,
has crashed.


Timeline
---------------------------------------------------------------------------------------------------

I discovered this issue in February of 2018, but tests on macOS High Sierra 10.13.4 Beta revealed
that the issue was already in the process of being fixed. I thus declined to report the issue to
Apple.


License
---------------------------------------------------------------------------------------------------

The xpc-crash code is released into the public domain. As a courtesy I ask that if you reference or
use any of this code you attribute it to me.


---------------------------------------------------------------------------------------------------
Brandon Azad
