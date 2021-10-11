---
layout: post
toc: true
title: "Thoughts on Primordia"
categories: programming
tags: [astroquery, python, astrophysics, data]
---

## Why, Though

Originally, I had a large list of astronomical objects from the ALLWISE data release.  Objects from ALLWISE contain a unique ID instead of a specific object type name, such as Star, Galaxy etc.  On the other hand, SIMBAD *does* contain these.  In other words, my purpose was to match my ALLWISE objects with the correpsonding objects from SIMBAD in order to give unique labels to my ALLWISE objects for classification purposes.

The ALLWISE object count greatly outnumbers the SIMBAD object count, so instead of using the website interface to match these objects, which may take weeks depending on how many objects you have, we can build a Python script which also takes an absurd amount of time to perform matches, but it's okay, because it's automated!

## Not Many of These Exist
In this tutorial, I'm going to be showing you how to build a Python script which queries objects and relevant information pertaining to those objects from the [SIMBAD](https://en.wikipedia.org/wiki/SIMBAD) and [ALLWISE](http://wise2.ipac.caltech.edu/docs/release/allwise/) astronomical databases.  

This Python script is going to be built using a tool for Python called [Astroquery](http://astroquery.readthedocs.io/en/latest/).

## Before We Begin
Make sure that Python is installed on your machine and you have the means to run Python programs.  For these Python scripts, I used Python v. 3.6.1

Additionally, make sure that Astroquery is installed.  Since I was developing these scripts using [Anaconda's Spyder IDE](https://pythonhosted.org/spyder/installation.html), I was able to run the following command in my Anaconda Prompt to install Astroquery:
```
conda install -c astropy astroquery
```

## Acquiring Matches from SIMBAD

At the top of the Python script, include the following imports that we will be using.

```python
import astropy
from astroquery.simbad import Simbad
from astropy.coordinates import SkyCoord
import astropy.units as u
import csv
import os
import glob
```

First thing is first - make sure you have a text file that contains the [RA, DEC] measurements of the objects that you wish to find matches for in SIMBAD.  In fact, you can have a lot of these text files in the same directory, because we are going to create code that loops through all of these text files and acquires matches.  Having only one is okay, too.

Make sure that the [RA, DEC] masurements are stored in the following format, where each line is a new position:
```
150.0000092,-63.5424157
150.0000372,-63.5000669
150.0000969,-64.1182345
150.0001572,-64.8274851
```

We will be storing the match results in a CSV file.  Create two path variables: one for the directory containing your [RA, DEC] measurements from WISE, the other a path to an output CSV file for the matches.  This is what mine looks like:

```python
WISE_positions = r'C:\Users\Joe\Documents\Astro_Research_2017\ALLWISE_RA_DEC'
outputfile = r'C:\Users\Joe\Documents\Astro_Research_2017\matches_original.csv'
```

Next, we are going to loop through all of the text files in the ```WISE_positions``` directory, storing all of the [RA, DEC] measurements into the list data structure called ```coordinates``` for future use.

```python
# loop through all RA and DEC text files in given directory
coordinates = []
path =  WISE_positions
for infile in glob.glob( os.path.join(path, '*.txt') ):
    # import RA and DEC measurements from WISE text file where measurements are separated by a comma
    with open(infile) as inputfile:
        for line in inputfile:
            coordinates.append(line.strip().split(','))
```

The next part is a bit tricky, because Astroquery sometimes returns results that are empty (i.e. no matches).  Additionally, the actual [RA, DEC] measurement may differ slightly in the SIMBAD results, depending on the specified matching radius, so let's account for this too.

We create three lists.  One stores the SIMBAD matches, including empty results, each of which will be a data structure unique to Astroquery, called a VOTable.  The next stores the SIMBAD matches that are not empty (which will require a strange boolean statement in the loop).  The last stores the correpsonding ALLWISE [RA, DEC] specific to that SIMBAD match.

```python
simbad_matches = []
real_match_data = [] #contains match results only, disregards "NoneType" i.e. no match
coordinate_save = [] #corresponding ALLWISE (RA,DEC) for SIMBAD match
```

Next, we loop through each row of our coordinate list, performing a query using this position measurement.  Before I give the full loop, let's dissect a few important pieces.

Take a look at the following line:

```python
 simbad_matches.append(Simbad.query_region(SkyCoord(coord[0], coord[1], unit=(u.deg, u.deg), frame='fk5'), radius = 1*u.arcsec))
```
To our list, we will be appending the result of querying a region from SIMBAD.  Our region is specified by Astropy coordinates, where we specify the units to be in degrees.  The frame we are working with is [FK5](http://docs.astropy.org/en/stable/api/astropy.coordinates.FK5.html).  I am trying to find matches that are within a radius of 1 arcsecond, but you can specify this to be whatever you want.  Take a look [here](http://astroquery.readthedocs.io/en/latest/simbad/simbad.html) for more options.

Unfortunately, I found that SIMBAD would sometimes return a result that would be empty.  To account for this, I made sure to include only results that were actual matches by checking if the match was the right type of data structure.  It turns out that a match takes the form of a "astropy.table.table.Table", so I included the following boolean:

```python
if(type(simbad_matches[i]) == astropy.table.table.Table)
```

The full loop should look like this:

```python
i = 0 #keeps track of index we want
for row in coordinates:
    coord = row # single RA and DEC measurement
    simbad_matches.append(Simbad.query_region(SkyCoord(coord[0], coord[1], unit=(u.deg, u.deg), frame='fk5'), radius = 1*u.arcsec))
    
    #discard results with no matches
    if(type(simbad_matches[i]) == astropy.table.table.Table): 
        real_match_data.append(simbad_matches[i])
        coordinate_save.append(coord) # save coordinates for future
    
    i+=1
```

Our two main lists of interest are now ```real_match_data```, which contains VOTable results at each index, and ```coordinate_save```, which contains the correpsonding WISE coordinates for each match.

We now need to extract what we really want from the VOTable results, which is the "MAIN_ID".  This will tell us the unique identifier for each object from the SIMBAD database.  This will be stored along with the correpsonding WISE RA and DEC.  Alternatively, you can also store the correpsonding SIMBAD RA and DEC instead.  I made sure to index those from the VOTable and comment out the lines needed to do that.

```python
#get only MAIN_ID, RA, and DEC from SIMBAD matches and store it in id_ra_dec list
id_ra_dec = []
j = 0
for table_result in real_match_data:
    temp = []
    decoded_id = table_result['MAIN_ID', 'RA', 'DEC'].as_array()[0][0].decode() #decode SIMBAD ID from byte type to string
    temp.append(decoded_id) # MAIN_ID
    temp.append(coordinate_save[j][0]) #RA
    temp.append(coordinate_save[j][1]) #DEC
    
    #these two lines are for storing the SIMBAD RA and DEC instead of the WISE ones
    #temp.append(table_result['MAIN_ID', 'RA', 'DEC'].as_array()[0][1]) #RA
    #temp.append(table_result['MAIN_ID', 'RA', 'DEC'].as_array()[0][2]) #DEC
    
    id_ra_dec.append(temp)
    j += 1
```

The line ```decoded_id = table_result['MAIN_ID', 'RA', 'DEC'].as_array()[0][0].decode()``` is what extracts the information from the matched VOTable.  As you can see, I only extracted "MAIN_ID", "RA", and "DEC", but feel free to extract whatever information you want to from the SIMBAD VOTable.  

Our new list, ```id_ra_dec``` contains what we were seeking in the first place.  The next step is to simply write each row of this to a CSV file.

```python
#write id_ra_dec to csv file
with open(outputfile, 'w') as f:
    writer = csv.writer(f)
    writer.writerows(id_ra_dec)
```

That's it!  [This](https://github.com/jjgccg/IRSA-ALLWISE-Classifier/blob/master/matching%20scripts/SIMBAD_Matcher.py) is the full Python script source code.

## Using Astroquery with WISE

While I'm here, I might as well explain how to build a similar Python script to scrape information from the ALLWISE database so you don't have to use the web interface to do so.

As before, we perform our imports and specify two directories: one which contains the [RA, DEC] measurements, and one which contains an output CSV file.  This time, my input file was the output CSV file from the previous tutorial.

```python
from astroquery.irsa import Irsa
from astropy.coordinates import SkyCoord
import astropy.units as u
import csv

SIMBAD_positions = r'C:\Users\jjgce\Desktop\SIMBAD_matches.csv' #path to dir containing SIMBAD ra and dec csv file
WISE_info_output = r'C:\Users\jjgce\Desktop\allwise_info.csv'
```

We first create a list and transfer the data from the input CSV file to this list.  As you can tell, I'm getting lazy with my decriptions at this point.

```python
obj_ra_dec = []
with open(SIMBAD_positions) as radec:
    readCSV = csv.reader(radec, delimiter=',')
    for row in readCSV:
        obj_ra_dec.append(row)

```

Below is the loop which extracts information from the ALLWISE database and stores the results into a list.

```python
# Using ra and dec from obj_ra_dec list, get WISE information for each object
# and append to WISE_info list
WISE_info = []
for row in obj_ra_dec:
    RA = float(row[1])
    DEC = float(row[2])
    query = Irsa.query_region(SkyCoord(RA, DEC, unit=(u.deg, u.deg), frame='fk5'), radius = 1*u.arcsec,catalog=('allwise_p3as_psd'))
    print(query)
    WISE_info.append(query)
```

As you can see in our line of code which performs the actual query, ```Irsa.query_region(SkyCoord(RA, DEC, unit=(u.deg, u.deg), frame='fk5'), radius = 1*u.arcsec,catalog=('allwise_p3as_psd'))```, we specify the catalog to be "allwise_p3as_psd".  This means that we want to gather results from the ALLWISE catalog instead of other IRSA catalogs.  More information about using Astroquery for IRSA can be found [here](http://astroquery.readthedocs.io/en/latest/irsa/irsa.html).  Again, we also make sure to perform matches within 1 arcsecond.

Now, we have a list of matches with all of the corresponding information for each match from the WISE database, including any extraneous information you might not want.  To get specific information from the results, we create a new loop where we decode specific information, and then append this to a new list.  You can see the information that I chose to index from each results, such as "w1mpro", "w4sigmpro", etc.  A full list of information you can extract can be found [here](http://irsa.ipac.caltech.edu/cgi-bin/Gator/nph-dd).

```python
# Get only specific information from each WISE object and append it to new list
WISE_specifics = []
for table_result in WISE_info:
    #edit indices of table_result to get info you want
    temp_info = table_result['w1mpro','w1sigmpro','w1snr','w1rchi2',
                             'w2mpro','w2sigmpro','w2snr','w2rchi2',
                             'w3mpro','w3sigmpro','w3snr','w3rchi2',
                             'w4mpro','w4sigmpro','w4snr','w4rchi2'].as_array()[0]
    WISE_specifics.append(temp_info)
```

Finally, we write this information to a CSV file.

```python
with open(WISE_info_output, 'w') as f:
    writer = csv.writer(f)
    writer.writerows(WISE_specifics)
```

See my github for full source code.
