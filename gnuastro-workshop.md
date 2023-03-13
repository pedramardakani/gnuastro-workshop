Gnuastro Workshop in Iran
--------------------------

### NoiseChisel and Multi-Extension FITS files

The first step is to separate the signal (galaxies or stars) from the background noise in the image. Gnuastro has NoiseChisel for this job. But NoiseChisel’s output is a multi-extension FITS file, therefore to better understand how to use NoiseChisel, let’s take a look at multi-extension FITS files and how you can interact with them.

In the FITS format, each extension contains a separate dataset (image in this case). You can get basic information about the extensions in a FITS file with Gnuastro’s Fits program.

To start with, let’s run NoiseChisel without any options, then use Gnuastro’s Fits program to inspect the number of extensions in this file.

```
$ astnoisechisel flat-ir/xdf-f160w.fits
$ astfits xdf-f160w_detected.fits
```

From the output list, we see that NoiseChisel’s output contains 5 extensions.

Metadata regarding how the analysis was done (or a dataset was created) is very important for higher-level analysis and reproducibility. Therefore, Let’s first take a closer look at the NOISECHISEL-CONFIG extension. If you specify a special header in the FITS file, Gnuastro’s Fits program will print the header keywords (metadata) of that extension. You can either specify the HDU/extension counter (starting from 0), or name. Therefore, the two commands below are identical for this file. We are usually tempted to use the first (shorter format), but when putting your commands into a script, please use the second format which is more human-friendly and understandable for readers of your code who may not know what is in the 0-th extension (this includes yourself in a few months!):

```
$ astfits xdf-f160w_detected.fits -h0
$ astfits xdf-f160w_detected.fits -hNOISECHISEL-CONFIG
```

The first group of `FITS` header keywords you see (containing the `SIMPLE` and `BITPIX` keywords; before the first empty line) are standard keywords. They are required by the FITS standard and must be present in any FITS extension. The second group starts with the input file name (value to the INPUT keyword). The rest of the keywords you see afterwards have the same name as NoiseChisel’s options, and the value used by NoiseChisel in this run is shown after the = sign. Finally, the last group (starting with `DATE`) contains the date and version information of Gnuastro and its dependencies that were used to generate this file. Besides the option values, these are also critical for future reproducibility of the result (you may update Gnuastro or its dependencies, and they may behave differently afterwards).
The “versions and date” group of keywords are present in all Gnuastro’s FITS extension outputs.

With the command below, let’s see what the default value of the --detgrowquant option is:

```
$ astnoisechisel -P | grep detgrowquant
```

To confirm that NoiseChisel used this value when we ran it above, let’s use grep to extract the keyword line with detgrowquant from the metadata extension. However, as you saw above, keyword names in the header is in all caps. So we need to ask grep to ignore case with the -i option.

```
$ astfits xdf-f160w_detected.fits -h0 | grep -i detgrowquant
```

In the output of the above command, you see HIERARCH at the start of the line. According to the FITS standard, HIERARCH is placed at the start of all keywords that have a name that is more than 8 characters long. Both the all-caps and the HIERARCH keyword can be annoying when you want to read/check the value. Therefore, the best solution is to use the --keyvalue option of Gnuastro’s astfits program as shown below. With it, you do not have to worry about HIERARCH or the case of the name (FITS keyword names are not case-sensitive).

```
$ astfits xdf-f160w_detected.fits -h0 --keyvalue=detgrowquant -q
```

The rest of the HDUs in NoiseChisel have data. So let’s open them in a DS9 window and then describe each:

```
$ astfits xdf-f160w_detected.fits
$ astscript-fits-view xdf-f160w_detected.fits
```

Just have in mind that NoiseChisel’s job is only detection (separating signal from noise). We will do segmentation on this result later to find the individual galaxies/peaks over the detected pixels.

The second extension of NoiseChisel’s output (numbered 1, named `INPUT-NO-SKY`) is the `Sky-subtracted` input that you provided. The third (`DETECTIONS`) is NoiseChisel’s main output which is a binary image with only two possible values for all pixels: 0 for noise and 1 for signal. Since it only has two values, to avoid taking too much space on your computer, its numeric datatype an unsigned 8-bit integer (or uint8)14. The fourth and fifth (`SKY` and `SKY_STD`) extensions, have the `Sky` and its standard deviation values for the input on a tile grid and were calculated over the undetected regions.

Each HDU/extension in a FITS file is an independent dataset (image or table) which you can delete from the FITS file, or copy/cut to another file. For example, with the command below, you can copy NoiseChisel’s DETECTIONS HDU/extension to another file:

```
$ astfits xdf-f160w_detected.fits --copy=DETECTIONS -odetections.fits
```

There are similar options to conveniently cut (--cut, copy, then remove from the input) or delete (--remove) HDUs from a FITS file also.

### NoiseChisel optimization for detection

So far we have obtained ran Noisechisel with default parameters.

One good way to see if you have missed any signal (small galaxies, or the wings of brighter galaxies) is to mask all the detected pixels and inspect the noise pixels. For this, you can use Gnuastro’s Arithmetic program.

The command below will produce mask-det.fits. In it, all the pixels in the INPUT-NO-SKY extension that are flagged 1 in the DETECTIONS extension (dominated by signal, not noise) will be set to NaN.

Since the various extensions are in the same file, for each dataset we need the file and extension name. To make the command easier to read/write/understand, let’s use shell variables: ‘in’ will be used for the Sky-subtracted input image and ‘det’ will be used for the detection map. Recall that a shell variable’s value can be retrieved by adding a $ before its name, also note that the double quotations are necessary when we have white-space characters in a variable value (like this case).

```
$ in="xdf-f160w_detected.fits -hINPUT-NO-SKY"
$ det="xdf-f160w_detected.fits -hDETECTIONS"
$ astarithmetic $in $det nan where --output=mask-det.fits
```

To invert the result (only keep the detected pixels), you can flip the detection map (from 0 to 1 and vice-versa) by adding a ‘not’ after the second $det:

```
$ astarithmetic $in $det not nan where --output=mask-sky.fits
```

Let’s tweak NoiseChisel’s configuration a little to get a better result on this dataset. Let’s check the overall detection process to get a better feeling of what NoiseChisel is doing with the following command.

```
$ astnoisechisel flat-ir/xdf-f160w.fits --checkdetection
```

The check images/tables are also multi-extension FITS files. As you saw from the command above, when check datasets are requested, NoiseChisel will not go to the end. It will abort as soon as all the extensions of the check image are ready. Please list the extensions of the output with astfits and then opening it with ds9 as we done above. If you have read the paper, you will see why there are so many extensions in the check image.

```
$ astfits xdf-f160w_detcheck.fits
$ astscript-fits-view xdf-f160w_detcheck.fits
```

Let’s focus on one step: the `OPENED_AND_LABELED` extension shows the initial detection step of NoiseChisel. We see the seeds of that correlated noise structure with many small detections (a relatively early stage in the processing). Such connections at the lowest surface brightness limits usually occur when the dataset is too smoothed, the threshold is too low, or the final “growth” is too much.
As you see from the 2nd (`CONVOLVED`) extension, the first operation that NoiseChisel does on the data is to slightly smooth it. However, the natural correlated noise of this dataset is already one level of artificial smoothing, so further smoothing it with the default kernel may be the culprit. To see the effect, let’s use a sharper kernel as a first step to convolve/smooth the input.

By default NoiseChisel uses a Gaussian with full-width-half-maximum (FWHM) of 2 pixels. We can use Gnuastro’s MakeProfiles to build a kernel with FWHM of 1.5 pixel (trun-cated at 5 times the FWHM, like the default) using the following command.


```
$ astmkprof --kernel=gaussian,1.5,5 --oversample=1
$ astnoisechisel flat-ir/xdf-f160w.fits --kernel=kernel.fits
$ astarithmetic $in $det nan where --output=mask-det.fits
```
Optimize the parameters.

```
astnoisechisel flat-ir/xdf-f160w.fits --kernel=kernel.fits \
               --noerodequant=0.95 --dthresh=0.1 --snquant=0.95
```

Create a directory for ketnel and move it to this directory.

```
$ mv kernel.fits kernel/
$ rm *.fits
```

We are now ready to finally run NoiseChisel on the three filters and keep the output in a dedicated directory (which we will call nc for simplicity).

```
$ rm *.fits
$ mkdir nc
$ for f in f105w f125w f160w; do \
   astnoisechisel flat-ir/xdf-$f.fits --output=nc/xdf-$f.fits \
                  --noerodequant=0.95 --dthresh=0.1 --snquant=0.95; \
  done
```

### NoiseChisel optimization for storage

Just mention about it.

### Segmentation and making a catalog

The main output of NoiseChisel is the binary detection map.  It only has two values: 1 or 0. This is useful when studying the noise or background properties, but hardly of any use when you actually want to study the targets/galaxies in the image, especially in such a deep field where almost everything is connected.

```
$ mkdir seg
$ astsegment nc/xdf-f160w.fits -oseg/xdf-f160w.fits
$ astsegment nc/xdf-f125w.fits -oseg/xdf-f125w.fits
$ astsegment nc/xdf-f105w.fits -oseg/xdf-f105w.fits
```

Segment’s operation is very much like NoiseChise. For example, the output is a multi-extension FITS file, it has check images and uses the undetected regions as a referenc. Please have a look at Segment’s multi-extension output to get a good feeling of what it has done. Do Not forget to flip through the extensions in the “Cube” window.

```
$ astscript-fits-view seg/xdf-f160w.fits
```
Like NoiseChisel, the first extension is the input. The CLUMPS extension shows the true “clumps” with values that are ≥ 1, and the diffuse regions labeled as −1. Please flip between the first extension and the clumps extension and zoom-in on some of the clumps to get a feeling of what they are. In the OBJECTS extension, we see that the large detections of NoiseChisel (that may have contained many galaxies) are now broken up into separate labels. Play with the color-bar and hover your mouse of the various detections to see their different labels.

The clumps are not affected by the hard-to-deblend and low signal-to-noise diffuse regions, they are more robust for calculating the colors (compared to objects). From this step onward, we will continue with clumps.
In other words, through MakeCatalog, you can “reduce” an image to a table (catalog of certain properties of objects
in the image). Each requested measurement (over each label) will be given a column in the output table. To see the full set of available measurements run it with `--help` like below (and scroll up), note that measurements are classified by context.

```
$ astmkcatalog --help
```

So let’s select the properties we want to measure in this tutorial. First of all, we need to know which measurement belongs to which object or clump, so we will start with the `--ids`.
We also want to measure (in this order) the Right Ascension (with `--ra`), Declination (`--dec`), magnitude (`--magnitude`), and signal-to-noise ratio (`--sn`) of the objects and clumps. Furthermore, as mentioned above, we also want measurements on clumps, so we also need to call `--clumpscat`. The following command will make these measurements on Segment’s F160W output and write them in a catalog for each object and clump in a FITS table.

```
$ mkdir cat
$ astmkcatalog seg/xdf-f160w.fits --ids --ra --dec --magnitude --sn \
               --zeropoint=25.94 --clumpscat --output=cat/xdf-f160w.fits
$ astmkcatalog seg/xdf-f125w.fits --ids --ra --dec --magnitude --sn \
               --zeropoint=26.23 --clumpscat --output=cat/xdf-f125w.fits
$ astmkcatalog seg/xdf-f105w.fits --ids --ra --dec --magnitude --sn \
               --zeropoint=26.27 --clumpscat --output=cat/xdf-f105w.fits
```

However, the galaxy properties might differ between the filters (which is the whole purpose behind observing in different filters!). Also, the noise properties and depth of the datasets differ. You can see the effect of these factors in the resulting clump catalogs, with Gnuastro’s Table program. We will go deep into working with tables in the next section, but in summary: the `-i` option will print information about the columns and number of rows. To see the column values, just remove the -i option. In the output of each command below, look at the Number of rows:, and note that they are different.

```
$ asttable cat/xdf-f105w.fits -hCLUMPS -i
$ asttable cat/xdf-f125w.fits -hCLUMPS -i
$ asttable cat/xdf-f160w.fits -hCLUMPS -i
```

An accurate color calculation can only be done when magnitudes are measured from the same pixels on all images and this can be done easily with MakeCatalog. In fact this is one of the reasons that NoiseChisel or Segment do not generate a catalog like most other detection/segmentation software. This gives you the freedom of selecting the pixels for measurement in any way you like (from other filters, other software, manually, etc.). Fortunately in these images, the Point spread function (PSF) is very similar, allowing us to use a single labeled image output for all filters17. The F160W image is deeper, thus providing better detection/segmentation, and redder, thus observing smaller/older stars and representing more of the mass in the galaxies. We will thus use the F160W filter as a reference and use its segment labels to identify which pixels to use for which objects/clumps. But we will do the measurements on the sky-subtracted F105W and F125W images (using MakeCatalog’s `--valuesfile` option) as shown below: Notice that the only difference between these calls and the call to generate the raw F160W catalog (excluding the zero point and the output name) is the `--valuesfile`.

```
$ astmkcatalog seg/xdf-f160w.fits --ids --ra --dec --magnitude --sn \
               --valuesfile=nc/xdf-f125w.fits --zeropoint=26.23 \
               --clumpscat --output=cat/xdf-f125w-on-f160w-lab.fits
$ astmkcatalog seg/xdf-f160w.fits --ids --ra --dec --magnitude --sn \
               --valuesfile=nc/xdf-f105w.fits --zeropoint=26.27 \
               --clumpscat --output=cat/xdf-f105w-on-f160w-lab.fits
```

After running the commands above, look into what MakeCatalog printed on the command-line. You can see that (as requested) the object and clump pixel labels in both were taken from the respective extensions in seg/xdf-f160w.fits. However, the pixel values and pixel Sky standard deviation were respectively taken from nc/xdf-f105w.fits and nc/xdf-f125w.fits. Since we used the same labeled image on all filters, the number of rows in both catalogs are now identical. Let’s have a look:

```
$ asttable cat/xdf-f105w-on-f160w-lab.fits -hCLUMPS -i
$ asttable cat/xdf-f125w-on-f160w-lab.fits -hCLUMPS -i
$ asttable cat/xdf-f160w.fits -hCLUMPS -i
```

### Measuring the dataset limits

Just mention to this section.

### Working with catalogs (estimating colors)

In the previous step we generated catalogs of objects and clumps over our dataset. The catalogs are available in the two extensions of the single FITS file. Let’s see the extensions and their basic properties with the Fits program:

```
$ astfits cat/xdf-f160w.fits
```

We should have used `-hOBJECTS` and `-hCLUMPS` instead of `-h1` and `-h2` respectively. The numbers are just used here to convey that both names or numbers are possible, in the next commands, we will just use names.

```
$ asttable cat/xdf-f160w.fits -h1 --info # Objects catalog info.
$ asttable cat/xdf-f160w.fits -h1 # Objects catalog columns.
$ asttable cat/xdf-f160w.fits -h2 -i # Clumps catalog info.
$ asttable cat/xdf-f160w.fits -h2 # Clumps catalog columns.
```

To print the contents of special column(s), just give the column number(s) (counting from 1) or the column name(s) (if they have one) to the --column (or -c) option. For example, if you just want the magnitude and signal-to-noise ratio of the clumps (in the clumps catalog), you can get it with any of the following commands.

```
$ asttable cat/xdf-f160w.fits -hCLUMPS --column=5,6
$ asttable cat/xdf-f160w.fits -hCLUMPS -c5,SN
$ asttable cat/xdf-f160w.fits -hCLUMPS -c5 -c6
$ asttable cat/xdf-f160w.fits -hCLUMPS -cMAGNITUDE -cSN
```

sing column names instead of numbers has many advantages:
- **1. You do not have to worry about the order of columns in the table.**
- **2. It acts as a documentation in the script.**
- **3. Column meta-data (including a name) are not just limited to FITS tables and can also be used in plain text tables.**

Table also has tools to limit the displayed rows. For example, with the first command below only rows with a magnitude in the range of 29 to 30 will be shown. With the second command, you can further limit the displayed rows to rows with an S/N larger than 10 (a range between 10 to infinity). You can further sort the output rows, only show the top (or bottom) N rows, etc.

```
$ asttable cat/xdf-f160w.fits -hCLUMPS --range=MAGNITUDE,28:29
$ asttable cat/xdf-f160w.fits -hCLUMPS \
           --range=MAGNITUDE,28:29 --range=SN,10:inf
```
Since cat/xdf-f160w.fits and cat/xdf-f105w-on-f160w-lab.fits have exactly the same number of rows and the rows correspond to the same clump, let’s merge them to have one table with magnitudes in both filters.
We can merge columns with the `--catcolumnfile` option like below. You give this option a file name (which is assumed to be a table that has the same number of rows as the main input), and all the table’s columns will be concatenated/appended to the main table. Now, try it out with the commands below. We will first look at the metadata of the first table (only the `CLUMPS` extension). With the second command, we will concatenate the two tables and write them in, two-in-one.fits and finally, we will check the new catalog’s metadata.

```
$ asttable cat/xdf-f160w.fits -i -hCLUMPS
$ asttable cat/xdf-f160w.fits -hCLUMPS --output=two-in-one.fits \
           --catcolumnfile=cat/xdf-f125w-on-f160w-lab.fits \
           --catcolumnhdu=CLUMPS
$ asttable two-in-one.fits -i
```

By comparing the two metadata, we see that both tables have the same number of rows. But what might have attracted your attention more, is that two-in-one.fits has double the number of columns (as expected, after all, you merged both tables into one file, and did not ask for any specific column). In fact you can concatenate any number of other tables in one command, for example:

```
$ asttable cat/xdf-f160w.fits -hCLUMPS --output=three-in-one.fits \
           --catcolumnfile=cat/xdf-f125w-on-f160w-lab.fits \
           --catcolumnfile=cat/xdf-f105w-on-f160w-lab.fits \
           --catcolumnhdu=CLUMPS --catcolumnhdu=CLUMPS
$ asttable three-in-one.fits -i
```

As you see, to avoid confusion in column names, Table has intentionally appended a -1 to the column names of the first concatenated table if the column names are already present, in the original table. For example, we have the original `RA` column, and another one called `RA-1`). Similarly a -2 has been added for the columns of the second concatenated table.

However, this example clearly shows a problem with this full concatenation: some columns are identical (for example, `HOST_OBJ_ID` and `HOST_OBJ_ID-1`), or not needed (for example, `RA-1` and `DEC-1` which are not necessary here). In such cases, you can use `--catcolumns` to only concatenate certain columns, not the whole table. For example, this
command:

```
$ asttable cat/xdf-f160w.fits -hCLUMPS --output=two-in-one-2.fits \
           --catcolumnfile=cat/xdf-f125w-on-f160w-lab.fits \
           --catcolumnhdu=CLUMPS --catcolumns=MAGNITUDE
$ asttable two-in-one-2.fits -i
```
You see that we have now only appended the MAGNITUDE column of cat/xdf-f125w-on-f160w-lab.fits. This is what we needed to be able to later subtract the magnitudes. Let’s go ahead and add the F105W magnitudes also with the command below. Note how we need to call `--catcolumnhdu` once for every table that should be appended, but we only call `--catcolumn` once (assuming all the tables that should be appended have this column).

```
$ asttable cat/xdf-f160w.fits -hCLUMPS --output=three-in-one-2.fits \
           --catcolumnfile=cat/xdf-f125w-on-f160w-lab.fits \
           --catcolumnfile=cat/xdf-f105w-on-f160w-lab.fits \
           --catcolumnhdu=CLUMPS --catcolumnhdu=CLUMPS \
           --catcolumns=MAGNITUDE
$ asttable three-in-one-2.fits -i
```

But we are not finished yet! There is a very big problem: it is not immediately clear which one of `MAGNITUDE`, `MAGNITUDE-1` or `MAGNITUDE-2` columns belong to which filter! Right now, you know this because you just ran this command. But in one hour, you’ll start doubting yourself and will be forced to go through your command history, trying to figure out if you added `F105W` first, or `F125W`. You should never torture your future-self (or your colleagues) like this! So, let’s rename these confusing columns in the matched catalog. Fortunately, with the --colmetadata option, you can correct the column metadata of the final table (just before it is written).
For example, by adding three calls of this option to the previous command, we write the filter name in the magnitude column name and description.

```
$ asttable cat/xdf-f160w.fits -hCLUMPS --output=three-in-one-3.fits \
           --catcolumnfile=cat/xdf-f125w-on-f160w-lab.fits \
           --catcolumnfile=cat/xdf-f105w-on-f160w-lab.fits \
           --catcolumnhdu=CLUMPS --catcolumnhdu=CLUMPS \
           --catcolumns=MAGNITUDE \
           --colmetadata=MAGNITUDE,MAG-F160W,log,"Magnitude in F160W." \
           --colmetadata=MAGNITUDE-1,MAG-F125W,log,"Magnitude in F125W." \
           --colmetadata=MAGNITUDE-2,MAG-F105W,log,"Magnitude in F105W."
$ asttable three-in-one-3.fits -i
```

We now have all three magnitudes in one table and can start doing arithmetic on them (to estimate colors, which are just a subtraction of magnitudes). To use column arithmetic, simply call the column selection option (`--column` or `-c`), put the value in single quotations and start the value with `arith` (followed by a space) like the example below. In column-arithmetic, you can identify columns by number (prefixed with a $) or name.

So let’s estimate one color from three-in-one-3.fits using column arithmetic. All the commands below will produce the same output, try them each and focus on the differences. Note that column arithmetic can be mixed with other ways to choose output columns (the `-c` option).

```
$ asttable three-in-one-3.fits -ocolor-cat.fits \
           -c1,2,3,4,'arith $5 $7 -'
$ asttable three-in-one-3.fits -ocolor-cat.fits \
           -c1,2,RA,DEC,'arith MAG-F125W MAG-F160W -'
$ asttable three-in-one-3.fits -ocolor-cat.fits -c1,2 \
           -cRA,DEC --column='arith MAG-F105W MAG-F160W -'
```

Have a look at the column metadata of the table produced above:

```
$ asttable color-cat.fits -i
```

The name of the column produced by arithmetic column is **ARITH_1**! This is natural: Arithmetic has no idea what the modified column is! You could have multiplied two columns, or done much more complex transformations with many columns. Metadata cannot be set automatically, your (the human) input is necessary. To add metadata, you can use `--colmetadata` like before:

```
$ asttable three-in-one-3.fits -ocolor-cat.fits -c1,2,RA,DEC \
           --column='arith MAG-F105W MAG-F160W -' \
           --colmetadata=ARITH_1,F105W-F160W,log,"Magnitude difference"
$ asttable color-cat.fits -i
```

We are now ready to make our final table. We want it to have the magnitudes in all three filters, as well as the three possible colors. Recall that by convention in astronomy colors are defined by subtracting the bluer magnitude from the redder magnitude. In this way a larger color value corresponds to a redder object. So from the three magnitudes, we can produce three colors (as shown below). Also, because this is the final table we are creating here and want to use it later, we will store it in cat/ and we will also give it a clear name and use the --range option to only print columns with a signal-to-noise ratio (SN column, from the F160W filter) above 5.

```
$ asttable three-in-one-3.fits --range=SN,5,inf -c1,2,RA,DEC,SN \
           -cMAG-F160W,MAG-F125W,MAG-F105W \
           -c'arith MAG-F125W MAG-F160W -' \
           -c'arith MAG-F105W MAG-F125W -' \
           -c'arith MAG-F105W MAG-F160W -' \
           --colmetadata=SN,SN-F160W,ratio,"F160W signal to noise ratio" \
           --colmetadata=ARITH_1,F125W-F160W,log,"Color F125W-F160W." \
           --colmetadata=ARITH_2,F105W-F125W,log,"Color F105W-F125W." \
           --colmetadata=ARITH_3,F105W-F160W,log,"Color F105W-F160W." \
           --output=cat/mags-with-color.fits
$ asttable cat/mags-with-color.fits -i
```

Let’s finish this section of the tutorial with a useful tip on modifying column metadata. Above, updating/changing column metadata was done with the `--colmetadata` in the same command that produced the newly created Table file. But in many situations, the table is already made and you just want to update the metadata of one column. In such cases
using `--colmetadata` is over-kill (wasting CPU/RAM energy or time if the table is large) because it will load the full table data and metadata into memory, just change the metadata and write it back into a file.
In scenarios when the table’s data does not need to be changed and you just want to set or update the metadata, it is much more efficient to use basic FITS keyword editing. For example, in the FITS standard, column names are stored in the **TTYPE** header keywords, so let’s have a look:

```
$ asttable two-in-one.fits -i
$ astfits two-in-one.fits -h1 | grep TTYPE
```

Changing/updating the column names is as easy as updating the values to these key-words. You do not need to touch the actual data! With the command below, we will just update the **MAGNITUDE** and **MAGNITUDE-1** columns (which are respectively stored in the **TTYPE5** and **TTYPE11** keywords) by modifying the keyword values and checking the effect by listing the column metadata again:

```
$ astfits two-in-one.fits -h1 \
          --update=TTYPE5,MAG-F160W \
          --update=TTYPE11,MAG-F125W
$ asttable two-in-one.fits -i
```

### Column statistics (color-magnitude diagram)

We created a single catalog containing the magnitudes of our desired clumps in all three filters, and their colors.
To start with, let’s inspect the distribution of three colors with the Statistics program.

```
$ aststatistics cat/mags-with-color.fits -cF105W-F125W
$ aststatistics cat/mags-with-color.fits -cF105W-F160W
$ aststatistics cat/mags-with-color.fits -cF125W-F160W
```

If you just want a specific measure, for example, the mean, median and standard deviation, you can ask for them specifically, like below:

```
$ aststatistics cat/mags-with-color.fits -cF105W-F160W \
                --mean --median --std
```

The basic statistics we measured above were just on one column. In many scenarios this is fine, but things get much more exciting if you look at the correlation of two columns with each other. For example, let’s create the color-magnitude diagram for our measured targets.

In many papers, the color-magnitude diagram is usually plotted as a scatter plot. However, scatter plots have a major limitation when there are a lot of points and they cluster together in one region of the plot: the possible correlation in that dense region is lost (because the points fall over each other). In such cases, it is much better to use a 2D histogram. In a 2D histogram, the full range in both columns is divided into discrete 2D bins (or pixels!) and we count how many objects fall in that 2D bin.

Since a 2D histogram is a pixelated space, we can simply save it as a FITS image and view it in a FITS viewer. Let’s do this in the command below. As is common with color-magnitude plots, we will put the redder magnitude on the horizontal axis and the color on the vertical axis. We will set both dimensions to have 100 bins (with `--numbins` for the horizontal and `--numbins2` for the vertical). Also, to avoid strong outliers in any of the dimensions, we will manually set the range of each dimension with the `--greaterequal`, `--greaterequal2`, `--lessthan` and `--lessthan2` options.

```
$ aststatistics cat/mags-with-color.fits -cMAG-F160W,F105W-F160W \
                --histogram2d=image --manualbinrange \
                --numbins=100 --greaterequal=22 --lessthan=30 \
                --numbins2=100 --greaterequal2=-1 --lessthan2=3 \
                --manualbinrange --output=cmd.fits
```

With the command below. Try hovering/zooming over the pixels: not only will you see the number of objects in catalog that fall in each bin/pixel, but you also see the F160W magnitude and color of that pixel also (in the same place you usually see RA and Dec when hovering over an astronomical image).

```
$ astscript-fits-view cmd.fits --ds9scale=minmax
```

Having a 2D histogram as a FITS image with WCS has many great advantages. For example, just like FITS images of the night sky, you can “match” many 2D histograms that were created independently. You can add two histograms with each other, or you can use advanced features of FITS viewers to find structure in the correlation of your columns. With the first command below, you can activate the grid feature of **DS9** to actually see the coordinate grid, as well as values on each line. With the second command, **DS9** will even read the labels of the axes and use them to generate an almost publication-ready plot.

```
$ astscript-fits-view cmd.fits --ds9scale=minmax --ds9extra="-grid yes"
$ astscript-fits-view cmd.fits --ds9scale=minmax \
           --ds9extra="-grid yes -grid type publication"
```

If you are happy with the grid and coloring and the rest, you can also use ds9 to save this as a JPEG image to directly use in your documents/slides with these extra DS9 options (DS9 will write the image to **cmd-2d.jpeg** and quit immediately afterwards):

```
$ astscript-fits-view cmd.fits --ds9scale=minmax \
           --ds9extra="-grid yes -grid type publication" \
           --ds9extra="-saveimage cmd-2d.jpeg -quit"
```

**Just mention the example exist in the tutorial**
This is good for a fast progress update. But for your paper or more official report, you want to show something with higher quality. For that, you can use the **PGFPlots** package in **LATEX**.
But to load the 2D histogram into PGFPlots first you need to convert the FITS image into a more standard format, for example, PDF. We will use Gnuastro for this, and use the sls-inverse color map (which will map the pixels with a value of zero to white):

```
$ astconvertt cmd.fits --colormap=sls-inverse --borderwidth=0 -ocmd.pdf
```

### Aperture photometry

The colors we calculated used a different segmentation map for each object. This might not satisfy some science cases that need the flux within a fixed area/aperture. Fortunately Gnuastro’s modular programs make it very easy do this type of measurement (photometry). To do this, we can ignore the labeled images of NoiseChisel of Segment, we can just built our own labeled image! That labeled image can then be given to `MakeCatalog`.
To generate the apertures catalog we will use Gnuastro’s MakeProfiles.

```
$ astmkprof -P
```

So we will first read the clump positions from the F160W catalog, then use AWK to set the other parameters of each profile to be a fixed circle of radius 5 pixels (recall that we want all apertures to have an identical size/area in this scenario).


```
$ asttable cat/xdf-f160w.fits -hCLUMPS -cRA,DEC \
           | awk '!/^#/{print NR, $1, $2, 5, 5, 0, 0, 1, NR, 1}' \
           > apertures.txt
$ cat apertures.txt
```

We can now feed this catalog into MakeProfiles using the command below to build the apertures over the image. The most important option for this particular job is `--mforflatpix`, it tells MakeProfiles that the values in the magnitude column should be used for each pixel of a flat profile. Without it, MakeProfiles would build the profiles such that the sum of the pixels of each profile would have a magnitude (in log-scale) of the value given in that column (what you would expect when simulating a galaxy for example).

```
$ astmkprof apertures.txt --background=flat-ir/xdf-f160w.fits \
            --clearcanvas --replace --type=int16 --mforflatpix \
            --mode=wcs --output=apertures.fits
```

Open apertures.fits with a FITS image viewer (like **SAO DS9**) and look around at the circles placed over the targets. Also open the input image and Segment’s clumps image and compare them with the positions of these circles. Where the apertures overlap, you will notice that one label has replaced the other (because of the `--replace` option). In the future, MakeCatalog will be able to work with overlapping labels, but currently it does not.

We can now feed the apertures.fits labeled image into MakeCatalog instead of Segment’s output as shown below. In comparison with the previous MakeCatalog call, you will notice that there is no more `--clumpscat` option, since there is no more separate “clump” image now, each aperture is treated as a separate “object”.

```
$ astmkcatalog apertures.fits -h1 --zeropoint=26.27 \
               --valuesfile=nc/xdf-f105w.fits \
               --ids --ra --dec --magnitude --sn \
               --output=cat/xdf-f105w-aper.fits
```

This catalog has the same number of rows as the catalog produced from clumps. Therefore similar to how we found colors, you can compare the aperture and clump magnitudes for example. You can also change the filter name and zero point magnitudes and run this command again to have the fixed aperture magnitude in the **F160W** filter and measure colors on apertures.

### Matching catalogs

In the example above, we had the luxury to generate the catalogs ourselves, and where thus able to generate them in a way that the rows match. But this is not generally the case. In many situations, you need to use catalogs from many different telescopes, or catalogs with high-level calculations that you cannot simply regenerate with the same pixels without spending a lot of time or using heavy computation. In such cases, when each catalog has the coordinates of its own objects, you can use the coordinates to match the rows with Gnuastro’s Match program.

As the name suggests, Gnuastro’s Match program will match rows based on distance (or aperture in 2D) in one, two, or three columns. For this tutorial, let’s try matching the two catalogs that were not created from the same labeled images, recall how each has a different number of rows:

```
$ asttable cat/xdf-f105w.fits -hCLUMPS -i
$ asttable cat/xdf-f160w.fits -hCLUMPS -i
```

You give Match two catalogs (from the two different filters we derived above) as argument, and the HDUs containing them (if they are FITS files) with the `--hdu` and `--hdu2` options. The `--ccol1` and `--ccol2` options specify the coordinate-columns which should be matched with which in the two catalogs. With `--aperture` you specify the acceptable error (radius in 2D), in the same units as the columns.

```
$ astmatch cat/xdf-f160w.fits cat/xdf-f105w.fits \
           --hdu=CLUMPS --hdu2=CLUMPS \
           --ccol1=RA,DEC --ccol2=RA,DEC \
           --aperture=0.5/3600 \
           --output=matched.fits
$ astfits matched.fits
```


From the second command, you see that the output has two extensions and that both have the same number of rows. The rows in each extension are the matched rows of the respective input table: those in the first HDU come from the first input and those in the second HDU come from the second. However, their order may be different from the input tables because the rows match: the first row in the first HDU matches with the first row in the second HDU, etc. You can also see which objects did not match with the `--notmatched`, like below. Note how each extension of now has a different number of rows.


```
$ astmatch cat/xdf-f160w.fits cat/xdf-f105w.fits \
           --hdu=CLUMPS --hdu2=CLUMPS \
           --ccol1=RA,DEC --ccol2=RA,DEC \
           --aperture=0.5/3600 \
           --output=not-matched.fits --notmatched
$ astfits not-matched.fits
```

The `--outcols` of Match is a very convenient feature: you can use it to specify which columns from the two catalogs you want in the output (merge two input catalogs into one). If the first character is an **a**, the respective matched column (number or name, similar to Table above) in the first catalog will be written in the output table. When the first character is a **b**, the respective column from the second catalog will be written in the output. Also, if the first character is followed by **_all**, then all the columns from the respective catalog will be put in the output.

```
$ astmatch cat/xdf-f160w.fits cat/xdf-f105w.fits \
           --hdu=CLUMPS --hdu2=CLUMPS \
           --ccol1=RA,DEC --ccol2=RA,DEC \
           --aperture=0.35/3600 \
           --outcols=a_all,bMAGNITUDE,bSN \
           --output=matched.fits
$ astfits matched.fits
```

### Reddest clumps, cutouts and parallelization

As a final step, let’s go back to the original clumps-based color measurement we generated. We will find the objects with the strongest color and make a cutout to inspect them visually and finally, we will see how they are located on the image. With the command below, we will select the reddest objects (those with a color larger than 1.5)

```
$ asttable cat/mags-with-color.fits --range=F105W-F160W,1.5,inf
```

Let’s crop the F160W image around each of these objects, but we first need a unique identifier for them. We will define this identifier using the object and clump labels (with an underscore between them) and feed the output of the command above to **AWK** to generate a catalog. Note that since we are making a plain text table, we will define the necessary (for the string-type first column) metadata manually.

```
$ echo "# Column 1: ID [name, str10] Object ID" > cat/reddest.txt
$ asttable cat/mags-with-color.fits --range=F105W-F160W,1.5,inf \
           | awk '{printf("%d_%-10d %f %f\n", $1, $2, $3, $4)}' \
           >> cat/reddest.txt
```

Let’s see how these objects are positioned over the dataset. DS9 has the “Region”s concept for this purpose. And you build such regions easily from a table using Gnuastro’s `astscript-ds9-region` installed script, using the command below:

```
astscript-ds9-region cat/reddest.txt -c2,3 --mode=wcs \
         --command="ds9 flat-ir/xdf-f160w.fits -zscale"
```

We can now feed **cat/reddest.txt** into Gnuastro’s Crop program to get separate postage stamps for each object. To keep things clean, we will make a directory called crop-red and ask Crop to save the crops in this directory. We will also add a **-f160w.fits** suffix to the crops (to remind us which filter they came from). The width of the crops will be 15 arc-seconds (or 15/3600 degrees, which is the units of the WCS).

```
$ mkdir crop-red
$ astcrop flat-ir/xdf-f160w.fits --mode=wcs --namecol=ID \
          --catalog=cat/reddest.txt --width=15/3600,15/3600 \
          --suffix=-f160w.fits --output=crop-red
```

If you look at the order of the crops, you will notice that the crops are not made in order! This is because each crop is independent of the rest, therefore crops are done in parallel, and parallel operations are asynchronous. So the order can differ in each run, but the final output is the same! In the command above, you can change **f160w** to **f105w** to make the crops in both filters. You can see all the cropped FITS files in the **crop-red** directory with this command:

```
$ astscript-fits-view crop-red/*.fits
```

### FITS images in a publication

### Marking objects for publication

### Writing scripts to automate the steps

### Citing and acknowledging Gnuastro
