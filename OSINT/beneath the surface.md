# Beneath the Surface

## Description

The threat actor `Mr0x00` has been observed leaving digital breadcrumbs across the surface web before disappearing into the dark. Intelligence suggests he maintains a public presence on under his known alias. Begin your hunt on the surface. Find him where people dump their secrets publicly. His alias is your first lead. What you find there will point you deeper?

## Writeup

The wording dumped their secrets publicly hinted towards public paste sites so with a little bit of fuzzing and trying different paste sites I found this [pastebin profile](https://pastebin.com/u/Mr0x00) which only had a single [paste](https://pastebin.com/4fXhRtBL) with the following content:

```md
You found me. Good. But this is just the shore — the real ocean is darker.
The flag doesn't live here. It lives where the internet forgets about you. Where people bury their secrets in encrypted bins that even the host can't read.
You already know the world above. Now find the world below.
The password to what's waiting: NotMuchSecure!
The ladder down: ?1e19d28759f16a16#8nmOODgWGbJFVsO/xwAU7xHuI8iwn3Pbgx9gE5czpSg=
The door? That's on you to find.
```

This gives us a backlink `?1e19d28759f16a16#8nmOODgWGbJFVsO/xwAU7xHuI8iwn3Pbgx9gE5czpSg=` with the password `NotMuchSecure!` on an "encrypted bin" paste. Given backlink was in a format similar to ?id#key matching the format of ZeroBin. The words "darker" and "world below" hinted towards upside dow- I mean the dark web so I tried to access the backlink on different .onion sites and found this [ZeroBin paste](http://zerobinftagjpeeebbvyzjcqyjpmjvynj5qlexwyxe7l3vqejxnqv5qd.onion/?1e19d28759f16a16#8nmOODgWGbJFVsO/xwAU7xHuI8iwn3Pbgx9gE5czpSg=) where I used the password and got the following content:

```md
Well Finally You are here!
It's an interesting place if you surf little.
it's became more interesting inside.

Flag : 
hackzero{y0u_c4m3_l0n6_w4y_5urf1n6!}
```

## Flag

`hackzero{y0u_c4m3_l0n6_w4y_5urf1n6!}`
