# Thread 0

## Description

The program starts. You open the binary. You set your breakpoint. You already missed it.

## Writeup

main is a decoy function and has no real relevance to the flag

```c
LABEL_12:
  if ( v14 )
    __rustc::__rust_dealloc(v12, 32 * v14, 8);
  if ( v2 && !(*(_DWORD *)v1 ^ 0x75626564 | *(unsigned __int8 *)(v1 + 4) ^ 0x67) )
    std::io::stdio::_print((unsigned __int8 *)"Debug mode not available in release build.\nSystem ready.\n", 0x57u);
  result = std::io::stdio::_print((unsigned __int8 *)"System ready.\n", 0x1Du);
  if ( v0 )
    return __rustc::__rust_dealloc(v1, v0, 1);
  return result;
```

So `main` is only checking for the string `debug` and then printing `System ready.`

The real flag logic is in `hackzero::tls_callback_fn` which IDA labeled code as `TlsCallback_0`.

```c
if ( a2 == 1 )
{
  std::env::args((__int64 *)&v21);
  <std::env::Args as core::iter::traits::iterator::Iterator>::next(&v23, (__int64)&v21);
```

That tells us that the code runs on process attach before normal execution reaches `main`. I also see:

```c
if ( v8 >= 2 )
{
    if ( *(_QWORD *)(v19 + 40) == 16 )
    {
        v9 = *(__m128 **)(v19 + 32);
```

So the password must be exactly 16 bytes. The actual validation is all in one compare chain:

```c
if ( 23 * __ROL1__(v9->m128_i8[0] ^ 0x5A, 1) == 106
    && 23 * __ROL1__(v9->m128_i8[1] ^ 0x91, 2) == 0xCA
    && 23 * __ROL1__(v9->m128_i8[2] ^ 0xC8, 3) == 52
    && 23 * __ROL1__(~v9->m128_i8[3], 4) == 38
    && 23 * __ROL1__(v9->m128_i8[4] ^ 0x36, 5) == 0xE2
    && 23 * __ROL1__(v9->m128_i8[5] ^ 0x6D, 6) == 117
    && 23 * __ROR1__(v9->m128_i8[6] ^ 0xA4, 1) == 0xBE
    && 23 * __ROL1__(v9->m128_i8[7] ^ 0xDB, 1) == 0xB5
    && 23 * __ROL1__(v9->m128_i8[8] ^ 0x12, 2) == 51
    && v9->m128_i8[9] == 72
    && v9->m128_i8[10] == 32
    && 23 * __ROL1__(v9->m128_i8[11] ^ 0xB7, 5) == 18
    && 23 * __ROL1__(v9->m128_i8[12] ^ 0xEE, 6) == 0xA2
    && 23 * __ROR1__(v9->m128_i8[13] ^ 0x25, 1) == 31
    && 23 * __ROL1__(v9->m128_i8[14] ^ 0x5C, 1) == 118
    && 23 * __ROL1__(v9->m128_i8[15] ^ 0x93, 2) == 38 )
```

I inverted those byte by byte. Since `23` is invertible mod `256` each condition can be solved directly.

That gives the password: `I USE ARCH BTW!!`

The success path right below confirms this is the real branch:

```c
{
  hackzero::decrypt_inner((__int64)&v14, v9);
  *(_QWORD *)&v16 = &v14;
  *((_QWORD *)&v16 + 1) = <alloc::string::String as core::fmt::Display>::fmt;
  std::io::stdio::_print(byte_14009B040, (unsigned __int64)&v16);
  if ( (_QWORD)v14 )
      __rustc::__rust_dealloc(*((_QWORD *)&v14 + 1), v14, 1);
  std::process::exit(0);
}
```

butttt solving dynamically is boring so I reversed the formation of the flag as well...

```c
v6 = *a2;
v7 = a2->m128_i8[0];
*v5 = _mm_xor_ps((__m128)xmmword_14009B000, *a2);
v5[1].m128_i8[0] = v7 ^ 0x7E;
v17 = v6;
v5[1].m128_i8[1] = v6.m128_i8[1] ^ 0x13;
v5[1].m128_i8[2] = v6.m128_i8[2] ^ 7;
```

```asm
.rdata:000000014009B000 xmmword_14009B000 xmmword 72151A0B09637C01637014060C60111Dh
```

So the first 16 output bytes are `password XOR xmmword_14009B000` and the last 3 bytes are `password[0] ^ 0x7E`, `password[1] ^ 0x13` and `password[2] ^ 7`

Using `I USE ARCH BTW!!` the decrypted text becomes `T15_C411B4CK_M4S73R` which just needs to be wrapped to form the flag.

## Flag

`hackzero{T15_C411B4CK_M4S73R}`
