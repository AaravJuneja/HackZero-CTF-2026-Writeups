# Who Clicked It First

## Description

A single dish photo is your only lead. Trace its earliest appearance online, identify who clicked it first and determine the exact date it was taken.

Flag Format: hackzero{name_dd-mm-yyyy}

## Writeup

TinEye gives me 4 direct hits for the image.
`https://www.tineye.com/search/0109391d1a4393ccecc82f846428e2c71aa1165b?tags=&sort=crawl_date&order=asc&page=1`

The earliest hit was May 14, 2016 but this is still not the oldest image so now using the complete image I reverse image searched it again. Using a higher quality image `https://img2.kochrezepte.at/use/7/holsteiner-schnitzel_7886.jpg` also did not yield a useful result.

By this info I concluded that it was not publicly indexed in main stream reverse image search engines. So instead I started searching for queries like "Holsteiner Schnitzel" and eventually ended up on the [flickr upload by keith kendall](https://www.flickr.com/photos/keithkendall/15880154310/) captured as part of his trip to The German cafe in Wilmington, NC on December 20, 2014.

## Flag

Using this we get the flag as `hackzero{keith_kendall_20-12-2014}`
