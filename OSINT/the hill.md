# The Hill

## Writeup

A major part of my view was blurred out and other details also ended up ragebaiting me, so I decided to use other techniques to solve this challenge.

On right clicking on the image, I found that the backend was based on Pannellum 2.5.6, and not the standard Google Maps API. My first instinct was to enumerate the metadata of the panorama to see if I could somehow get a clue about the location there. However, since the panorama was just a DOM view and not a standard image, I decided to look in the network requests for other images. From there, I managed to extract [streetview-2.jpg](https://vantage.vitbctf.dev/streetview-2.jpg). The version I extracted is [streetview-2.jpg](../images/streetview-2.jpg).

After this, I ran `exiftool` on the image to see if anything about the location (coordinates, city, etc.) had been leaked in the metadata. Although there was no information about the location itself, I found some *interesting* fields there:

```yaml
Software                        : Street View Download 360
Image ID                        : 7boNaDRJ3rurxD0XURpAcA
Copyright                       : Google
Image Number                    : 0
Warning                         : Invalid EXIF text encoding for UserComment
User Comment                    : Downloaded with Street View Download 360. Visit svd360.com for more info. Panorama ID: 7boNaDRJ3rurxD0XURpAcA
```

The Panorama ID is a unique ID used by the Google Maps API to identify every panorama in its Street View functionality. Using this ID, I was able to reconstruct the URL in Google Maps Street View that points to this exact panorama: `https://www.google.com/maps/@?api=1&map_action=pano&pano=7boNaDRJ3rurxD0XURpAcA`.

Now that we're literally in the Street View for the exact location provided in the challenge, we can simply move one step forward and then one step back. By doing this, the exact coordinates of the location show up in the URL shown in the browser. The coordinates reflected there were @18.7726386,82.7199468. I simply entered these in the challenge website and got the flag.

After doing this challenge, I realized that this method was actually a bug that could be applied to all 5 of the GeoOSINT challenges, but by the time I got to know about it, I'd already solved the other four :P

## Flag

`hackzero{y0u_4r3_1n_r3m073_0dd1sh4}`
