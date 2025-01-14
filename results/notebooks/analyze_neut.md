
<h1>Table of Contents<span class="tocSkip"></span></h1>
<div class="toc"><ul class="toc-item"><li><span><a href="#Fit-and-plot-neutralization-curves" data-toc-modified-id="Fit-and-plot-neutralization-curves-1">Fit and plot neutralization curves</a></span><ul class="toc-item"><li><span><a href="#Import-Python-modules-/-packages" data-toc-modified-id="Import-Python-modules-/-packages-1.1">Import Python modules / packages</a></span></li><li><span><a href="#Configuration-and-setup" data-toc-modified-id="Configuration-and-setup-1.2">Configuration and setup</a></span></li><li><span><a href="#Read-neutralization-data" data-toc-modified-id="Read-neutralization-data-1.3">Read neutralization data</a></span></li><li><span><a href="#Get-serum-concentrations" data-toc-modified-id="Get-serum-concentrations-1.4">Get serum concentrations</a></span></li><li><span><a href="#Fit-and-plot-all-neutralization-curves" data-toc-modified-id="Fit-and-plot-all-neutralization-curves-1.5">Fit and plot all neutralization curves</a></span></li><li><span><a href="#Neutralization-curves-integrated-with-logo-plots" data-toc-modified-id="Neutralization-curves-integrated-with-logo-plots-1.6">Neutralization curves integrated with logo plots</a></span></li></ul></li></ul></div>

# Fit and plot neutralization curves
In this notebook we will plot neutralization curves from GFP-based neutralization assays. 
The GFP-based neutralization assay system is described in detail [here](https://github.com/jbloomlab/flu_PB1flank-GFP_neut_assay).

All the curves plotted here represent the mean and standard deviation of three replicates, with each replicate in a separate column of a 96-well plate.

The curves fit are the Hill-style neutralization functions fit and plotted by the [neutcurve](https://jbloomlab.github.io/neutcurve/) package.

## Import Python modules / packages
In addition to importing the [neutcurve](https://jbloomlab.github.io/neutcurve/) package, we import [svgutils](https://svgutils.readthedocs.io) to merge the neutralization curves with the logo plots, and [cairosvg](https://cairosvg.org/) to convert SVGs to other formats:


```python
import collections
import itertools
import os
import re
import warnings
import xml.etree.ElementTree as ElementTree

import cairosvg

from IPython.display import display, HTML

import matplotlib.pyplot as plt

import numpy

import pandas as pd

import svgutils.compose

import yaml

from dms_tools2.ipython_utils import showPDF

import neutcurve
from neutcurve.colorschemes import CBPALETTE
import neutcurve.parse_excel

print(f"Using neutcurve version {neutcurve.__version__}")
```

    Using neutcurve version 0.3.0


Interactive matplotlib plotting:


```python
plt.ion()
```

Suppress warnings that can clutter output:


```python
warnings.simplefilter('ignore')
```

## Configuration and setup
Read general configuration from [config.yaml](config.yaml), which in turn specifies additional configuration files:


```python
with open('config.yaml') as f:
    config = yaml.safe_load(f)
```

Read the neutralization assay configuration from the specified file:


```python
print(f"Reading neutralization assay setup from {config['neut_config']}")

with open(config['neut_config']) as f:
    neut_config = yaml.safe_load(f)
```

    Reading neutralization assay setup from data/neut_assays/neut_config.yaml


Get the output directory:


```python
outdir = config['neutresultsdir']
os.makedirs(outdir, exist_ok=True)
print(f"Output will be written to {outdir}")
```

    Output will be written to results/neutralization_assays


## Read neutralization data

Next, for each dict in *neut_config*, we use [neutcurve.parse_excel.parseRachelStyle2019](https://jbloomlab.github.io/neutcurve/neutcurve.parse_excel.html#neutcurve.parse_excel.parseRachelStyle2019) to parse the raw Excel files to create a tidy data frame appropriate for passing to [neutcurve.CurveFits](https://jbloomlab.github.io/neutcurve/neutcurve.curvefits.html#neutcurve.curvefits.CurveFits). 
We then concatenate all the tidy data frames to get our neutralization data.
This is essentially the workflow [explained here](https://jbloomlab.github.io/neutcurve/rachelstyle2019_example.html):


```python
neutdata = []  # store all data frame, then concatenate at end

for sampledict in neut_config:
    assert len(sampledict) == 1
    sampleset, kwargs = list(sampledict.items())[0]
    print(f"Parsing data for {sampleset}...")
    neutdata.append(neutcurve.parse_excel.parseRachelStyle2019(**kwargs))

neutdata = pd.concat(neutdata)
print(f"Read data for {len(neutdata.groupby('serum'))} sera and "
      f"{len(neutdata.groupby(['serum', 'virus']))} serum / virus pairs.")
```

    Parsing data for VIDD1...
    Parsing data for VIDD2...
    Parsing data for VIDD3...
    Parsing data for VIDD4...
    Parsing data for VIDD5...
    Parsing data for 557v1...
    Parsing data for 557v2...
    Parsing data for 574v1...
    Parsing data for 574v2...
    Parsing data for 589v1...
    Parsing data for 589v2...
    Parsing data for 571v1...
    Parsing data for 571v2...
    Parsing data for ferret-Pitt-1-preinf...
    Parsing data for ferret-Pitt-1-postinf...
    Parsing data for ferret-Pitt-2-preinf...
    Parsing data for ferret-Pitt-2-postinf...
    Parsing data for ferret-Pitt-3-preinf...
    Parsing data for ferret-Pitt-3-postinf...
    Parsing data for ferret-WHO-Perth2009...
    Parsing data for ferret-WHO-Victoria2011...
    Parsing data for antibody-5A01...
    Parsing data for antibody-3C06...
    Parsing data for antibody-3C04...
    Parsing data for antibody-4C01...
    Parsing data for antibody-4F03...
    Parsing data for antibody-1C04...
    Read data for 27 sera and 119 serum / virus pairs.


These data look like this:


```python
display(HTML(neutdata.head().to_html(index=False)))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>serum</th>
      <th>virus</th>
      <th>replicate</th>
      <th>concentration</th>
      <th>fraction infectivity</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2010-age-21</td>
      <td>wt</td>
      <td>1</td>
      <td>0.000024</td>
      <td>1.010575</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>wt</td>
      <td>1</td>
      <td>0.000046</td>
      <td>0.981598</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>wt</td>
      <td>1</td>
      <td>0.000086</td>
      <td>0.997023</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>wt</td>
      <td>1</td>
      <td>0.000163</td>
      <td>0.954407</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>wt</td>
      <td>1</td>
      <td>0.000308</td>
      <td>1.005039</td>
    </tr>
  </tbody>
</table>


Write the neutralization data to a CSV file in our output directory:


```python
neutdatafile = os.path.join(outdir, 'neutdata.csv')
neutdata.to_csv(neutdatafile, index=False)
print(f"Wrote neutralization data to {neutdatafile}")
```

    Wrote neutralization data to results/neutralization_assays/neutdata.csv


## Get serum concentrations
We get the mean concentration over the three replicates:


```python
mean_concentration = (
 pd.read_csv(os.path.join(config['selectiontabledir'], 'serum_dilution_table.csv'))
 .rename(columns={'serum_name_formatted': 'serum'})
 .drop(columns='serum_group')
 .query('not serum.str.contains("antibody")')
 .set_index('serum')
 .apply(pd.to_numeric, errors='coerce')
 .mean(axis=1, numeric_only=True)
 .dropna()
 .rename('concentration')
 .reset_index()
 )

display(HTML(mean_concentration.to_html(index=False)))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>serum</th>
      <th>concentration</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>0.005833</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>0.000750</td>
    </tr>
    <tr>
      <td>2015-age-29-prevacc</td>
      <td>0.015333</td>
    </tr>
    <tr>
      <td>2015-age-29-vacc</td>
      <td>0.001208</td>
    </tr>
    <tr>
      <td>2015-age-48-prevacc</td>
      <td>0.017833</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>0.000542</td>
    </tr>
    <tr>
      <td>2015-age-49-prevacc</td>
      <td>0.018500</td>
    </tr>
    <tr>
      <td>2015-age-49-vacc</td>
      <td>0.011167</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>0.005417</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>0.006083</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>0.003142</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>0.002083</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>0.009833</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>0.000433</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-preinf</td>
      <td>0.000750</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>0.001458</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-preinf</td>
      <td>0.002000</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>0.001792</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-preinf</td>
      <td>0.004917</td>
    </tr>
    <tr>
      <td>ferret-WHO-Perth2009</td>
      <td>0.011167</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>0.003750</td>
    </tr>
  </tbody>
</table>


Now convert to dict appropriate for passing to `neutcurve.CurveFits.plotSera` as `vlines` parameter:


```python
vlines = {tup.serum: [{'x': tup.concentration}] for tup in
          mean_concentration.itertuples(index=False)}
```

## Fit and plot all neutralization curves

Now we fit the neutralization curves with a `neutcurve.CurveFits`:


```python
fits = neutcurve.CurveFits(neutdata)
```

Make a big panel of plots of the across-replicate averages for all sera and antibodies, drawing a vertical line at the mean serum concentration used in the mutational antigenic profiling of the sera:


```python
for is_antibody, ptype, xlabel in [(False, 'sera', 'serum dilution'),
                                   (True, 'antibody', 'concentration ($\mu$g/ml)')]:
    plotfile = os.path.join(outdir, f"all_{ptype}_plots.pdf")
    print(f"\nPlotting all {ptype} curves to {plotfile}...")
    fig, _ = fits.plotSera(
                sera=[s for s in fits.sera if ('antibody' in s) == is_antibody],
                xlabel=xlabel,
                max_viruses_per_subplot=6,
                vlines=vlines,
                )
    display(fig)
    fig.savefig(plotfile)
    plt.close(fig)
```

    
    Plotting all sera curves to results/neutralization_assays/all_sera_plots.pdf...



![png](analyze_neut_files/analyze_neut_27_1.png)


    
    Plotting all antibody curves to results/neutralization_assays/all_antibody_plots.pdf...



![png](analyze_neut_files/analyze_neut_27_3.png)


Now get the curve fit parameters (e.g., IC50s):


```python
fitparams = fits.fitParams()
```

Here is a list of all of the IC50s for the replicate-averages for each serum / virus pair.
Note that for sera there are dilutions, and for antibodies that are $\mu$g/ml:


```python
display(HTML(fitparams
             .query('replicate == "average"')
             .drop(columns=['midpoint', 'top', 'bottom', 'replicate',
                            'ic50', 'ic50_bound', 'slope', 'nreplicates'])
             .to_html(index=False)
             ))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>serum</th>
      <th>virus</th>
      <th>ic50_str</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2010-age-21</td>
      <td>wt</td>
      <td>0.00118</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>F193D</td>
      <td>&gt;0.014</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>syn</td>
      <td>0.00164</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>K189D</td>
      <td>0.0018</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>L157D</td>
      <td>0.00107</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>wt2</td>
      <td>0.00148</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>F159G</td>
      <td>0.00148</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>K160T</td>
      <td>0.00329</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>wt</td>
      <td>0.00124</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>L157D</td>
      <td>0.00481</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>K160T</td>
      <td>0.00338</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>F193D</td>
      <td>0.00234</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>syn</td>
      <td>0.00153</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>wt2</td>
      <td>0.00144</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>K189D</td>
      <td>0.000548</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>F159G</td>
      <td>0.00122</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>wt</td>
      <td>0.00172</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>L157D</td>
      <td>0.00931</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>K160T</td>
      <td>0.00387</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>F193D</td>
      <td>0.00418</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>F159G</td>
      <td>0.00174</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>wt</td>
      <td>0.000318</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>F159G</td>
      <td>0.0108</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>K189D</td>
      <td>0.000258</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>F193D</td>
      <td>0.000386</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>wt2</td>
      <td>0.000231</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>K160T</td>
      <td>0.000649</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>L157D</td>
      <td>0.000213</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>syn</td>
      <td>0.000386</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>wt</td>
      <td>0.000328</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>F193D</td>
      <td>&gt;0.0109</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>K160T</td>
      <td>0.00145</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>syn</td>
      <td>0.000412</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>N121E</td>
      <td>0.000332</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>F159G</td>
      <td>0.000805</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>wt2</td>
      <td>0.000192</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>L157D</td>
      <td>0.00027</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>K189D</td>
      <td>0.000202</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>wt</td>
      <td>0.00126</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>F159G</td>
      <td>&gt;0.0102</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>R220D</td>
      <td>&gt;0.0102</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>K189D</td>
      <td>0.000868</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>wt-2</td>
      <td>0.00109</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>F159G-2</td>
      <td>&gt;0.0102</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>R220D-2</td>
      <td>&gt;0.0102</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>wt</td>
      <td>8.54e-05</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>F159G</td>
      <td>&gt;0.00117</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>syn</td>
      <td>0.000141</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>wt2</td>
      <td>0.000113</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>R220D</td>
      <td>0.000141</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>K189D</td>
      <td>0.000132</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>wt-2</td>
      <td>0.000114</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>F159G-2</td>
      <td>&gt;0.00117</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>R220D-2</td>
      <td>0.00014</td>
    </tr>
    <tr>
      <td>2015-age-29-prevacc</td>
      <td>wt</td>
      <td>0.00637</td>
    </tr>
    <tr>
      <td>2015-age-29-prevacc</td>
      <td>F159G</td>
      <td>0.00954</td>
    </tr>
    <tr>
      <td>2015-age-29-prevacc</td>
      <td>K144E</td>
      <td>0.00561</td>
    </tr>
    <tr>
      <td>2015-age-29-vacc</td>
      <td>wt</td>
      <td>0.000299</td>
    </tr>
    <tr>
      <td>2015-age-29-vacc</td>
      <td>K144E</td>
      <td>0.000559</td>
    </tr>
    <tr>
      <td>2015-age-29-vacc</td>
      <td>F159G</td>
      <td>0.000293</td>
    </tr>
    <tr>
      <td>2015-age-48-prevacc</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>wt</td>
      <td>8.33e-05</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>K189D</td>
      <td>0.000413</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>F159G</td>
      <td>0.000149</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>R220D</td>
      <td>5.53e-06</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>K144E</td>
      <td>7.23e-05</td>
    </tr>
    <tr>
      <td>2015-age-49-prevacc</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
    </tr>
    <tr>
      <td>2015-age-49-vacc</td>
      <td>wt</td>
      <td>0.00343</td>
    </tr>
    <tr>
      <td>2015-age-49-vacc</td>
      <td>G75(HA2)H</td>
      <td>0.00277</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-preinf</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>wt</td>
      <td>9.84e-05</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>K189D</td>
      <td>0.000292</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>F193D</td>
      <td>0.000484</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>syn</td>
      <td>9.9e-05</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>F159G</td>
      <td>0.000186</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>K144E</td>
      <td>0.000176</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-preinf</td>
      <td>wt</td>
      <td>&gt;0.00206</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>wt</td>
      <td>0.000343</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>K189D</td>
      <td>0.000724</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>F193D</td>
      <td>0.0012</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>K144E</td>
      <td>0.000636</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>wt2</td>
      <td>0.000387</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>R220D</td>
      <td>4.65e-05</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>L157D</td>
      <td>0.000486</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>F159G</td>
      <td>0.000341</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-preinf</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>wt</td>
      <td>0.000375</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>K189D</td>
      <td>0.00113</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>F193D</td>
      <td>0.00505</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>K160T</td>
      <td>0.000471</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>L157D</td>
      <td>0.000265</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>wt2</td>
      <td>0.000732</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>R220D</td>
      <td>8.47e-05</td>
    </tr>
    <tr>
      <td>ferret-WHO-Perth2009</td>
      <td>wt</td>
      <td>0.00476</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>wt</td>
      <td>0.00192</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>K189D</td>
      <td>0.0115</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>F193D</td>
      <td>&gt;0.0125</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>F159G</td>
      <td>0.00601</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>K160T</td>
      <td>0.00871</td>
    </tr>
    <tr>
      <td>antibody-5A01</td>
      <td>wt</td>
      <td>0.0884</td>
    </tr>
    <tr>
      <td>antibody-5A01</td>
      <td>F159G</td>
      <td>&gt;1</td>
    </tr>
    <tr>
      <td>antibody-5A01</td>
      <td>syn</td>
      <td>0.142</td>
    </tr>
    <tr>
      <td>antibody-5A01</td>
      <td>K160T</td>
      <td>0.549</td>
    </tr>
    <tr>
      <td>antibody-3C06</td>
      <td>wt</td>
      <td>0.00726</td>
    </tr>
    <tr>
      <td>antibody-3C06</td>
      <td>F159G</td>
      <td>&gt;0.5</td>
    </tr>
    <tr>
      <td>antibody-3C06</td>
      <td>K160T</td>
      <td>0.0269</td>
    </tr>
    <tr>
      <td>antibody-3C04</td>
      <td>wt</td>
      <td>0.00774</td>
    </tr>
    <tr>
      <td>antibody-3C04</td>
      <td>K160T</td>
      <td>&gt;0.5</td>
    </tr>
    <tr>
      <td>antibody-3C04</td>
      <td>K160S</td>
      <td>0.333</td>
    </tr>
    <tr>
      <td>antibody-4C01</td>
      <td>wt</td>
      <td>0.0207</td>
    </tr>
    <tr>
      <td>antibody-4C01</td>
      <td>F193D</td>
      <td>&gt;0.5</td>
    </tr>
    <tr>
      <td>antibody-4C01</td>
      <td>syn</td>
      <td>0.0112</td>
    </tr>
    <tr>
      <td>antibody-4C01</td>
      <td>K160T</td>
      <td>0.0202</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>wt</td>
      <td>0.362</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>N121E</td>
      <td>&gt;3</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>F193D</td>
      <td>0.13</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>F159G</td>
      <td>0.0649</td>
    </tr>
    <tr>
      <td>antibody-1C04</td>
      <td>wt</td>
      <td>0.0868</td>
    </tr>
    <tr>
      <td>antibody-1C04</td>
      <td>K82A</td>
      <td>&gt;5</td>
    </tr>
  </tbody>
</table>


Write all of the fit parameters (including IC50s) for all replicates to a file:


```python
fitfile = os.path.join(outdir, 'fitparams.csv')
print(f"Writing fit parameters to {fitfile}")
fitparams.to_csv(fitfile, index=False, float_format='%.3g')
```

    Writing fit parameters to results/neutralization_assays/fitparams.csv


## Neutralization curves integrated with logo plots
Now we are going to make versions of the neutralization curves that are combined with the logo plots created by [analyze_map.ipynb](analyze_map.ipynb) to serve as figures.

Get the colors that we have specified for viruses in each serum group for these figures:


```python
with open(config['figure_config']) as f:
    figure_config = yaml.safe_load(f)
```

Get data frame with all sera for each figure:


```python
sera_df = pd.concat(
           [pd.DataFrame({'figure': figure,
                          'sera': figure_d['sera']})
            for figure, figure_d in figure_config['figures'].items()
            ],
           ignore_index=True
           )
```

Now draw the plots as single-column figures that can be aligned with the logo plots and save them as SVGs:


```python
sera_viruses_plotted = [] # list 2-tuples of serum / viruses plotted
neutsvgfiles = []
for figure, df in sera_df.groupby('figure'):
    
    ifigconfig = figure_config['figures'][figure]
    colors = ifigconfig['colors']
    sera = [s for s in df['sera'].unique() if s in fits.sera]
    xlabel = ['concentration ($\mu$g/ml)' if 'antibody' in s else 'serum dilution' for s in sera]
    
    # do we have one or multiple x-axis labels, which determines whether we try to align to logos
    if len(set(xlabel)) == 1:
        xlabel = xlabel[0]
        align_to_dmslogo_facet={'height_per_ax': 2.5,
                                'hspace': 0.8,
                                'tmargin': 0.4,
                                'bmargin': 1.3,
                                'right': 0.75,
                                'left': 0.2,
                                }
        heightscale = 1
        widthscale = 1.37
        sharex = True
    else:
        align_to_dmslogo_facet = False
        heightscale = 1.23
        widthscale = 1.1
        sharex = False
    
    viruses = list(colors.keys())
    ignore_serum_virus = (ifigconfig['ignore_serum_virus'] if
                          'ignore_serum_virus' in ifigconfig else None)
    sera_viruses_plotted += [(s, v) for s, v in itertools.product(sera, viruses)
                             if not ignore_serum_virus or
                             s not in ignore_serum_virus or
                             v not in ignore_serum_virus[s]]
    
    fig, _ = fits.plotSera(
                sera=sera,
                viruses=viruses,
                ignore_serum_virus=ignore_serum_virus,
                titles=[ifigconfig['sera_names'][s] for s in sera] if 'sera_names' in ifigconfig
                        else [s.replace('-', ' ') for s in sera],
                virus_to_color_marker=colors,
                xlabel=xlabel,
                max_viruses_per_subplot=len(colors),
                ncol=1,
                titlesize=17,
                labelsize=17,
                legendfontsize=14,
                ticksize=12,
                widthscale=widthscale,
                heightscale=heightscale,
                markersize=7,
                align_to_dmslogo_facet=align_to_dmslogo_facet,
                despine=True,
                yticklocs=[0, 0.5, 1],
                sharex=sharex,
                vlines=None if 'spikein' in figure else vlines,
                )
    
    neutsvgfile = os.path.join(config['figsdir'], f"{figure}_neut.svg")
    print(f"Saving plot for {figure} to {neutsvgfile}")
    if sharex:
        fig.savefig(neutsvgfile)
    else:
        fig.savefig(neutsvgfile, bbox_inches='tight')
    plt.close(fig)
    neutsvgfiles.append(neutsvgfile)
```

    Saving plot for 2009_age_53_samples to results/figures/2009_age_53_samples_neut.svg
    Saving plot for Hensley_sera to results/figures/Hensley_sera_neut.svg
    Saving plot for VIDD_sera to results/figures/VIDD_sera_neut.svg
    Saving plot for antibody_lower_head to results/figures/antibody_lower_head_neut.svg
    Saving plot for antibody_region_B to results/figures/antibody_region_B_neut.svg
    Saving plot for antibody_spikein to results/figures/antibody_spikein_neut.svg
    Saving plot for ferret to results/figures/ferret_neut.svg


Now we will combine the logo and SVG files. 

First, we need to define a function to get the width / height of a SVG file in points (see [here](http://osgeo-org.1560.x6.nabble.com/Get-size-of-SVG-in-Python-td5273032.html)):


```python
def svg_dim(svgfile, dim):
    """Get width or height `dim` of `svgfile` in points."""
    return float(ElementTree.parse(svgfile).getroot().attrib[dim].replace('pt', ''))
```

We also define a function that converts a SVG into a PDF or PNG:


```python
def svg_to_pdf(svgfile):
    """`svgfile` to PDF, return converted file name."""
    with open(svgfile) as f:
        svg = f.read()
    # need to eliminate units that `svgutils` incorrectly puts in viewBox
    viewbox_match = re.compile('viewBox="' + ' '.join(['\d+\.{0,1}\d*(px){0,1}'] * 4) + '"')
    if len(viewbox_match.findall(svg)) != 1:
        raise ValueError(f"did not find exactly one viewBox in {svgfile}")
    viewbox = viewbox_match.search(svg).group(0)
    svg = svg.replace(viewbox, viewbox.replace('px', ''))
    outfile = os.path.splitext(svgfile)[0] + '.pdf'
    cairosvg.svg2pdf(bytestring=svg, write_to=outfile)
    return outfile
```

Now we combine the neutralization curve SVGs with the logo-plot SVGs (which should already exist as output of [analyze_map.ipynb](analyze_map.ipynb)) as [here](https://svgutils.readthedocs.io/en/latest/tutorials/composing_multipanel_figures.html):


```python
for neutsvgfile in neutsvgfiles:
    logosvgfile = neutsvgfile.replace('_neut', '_logo')
    assert os.path.isfile(logosvgfile), f"cannot find {logosvgfile}"
    svgfile = neutsvgfile.replace('_neut', '_logo_and_neut')
    plots = [logosvgfile, neutsvgfile]
    widths = [svg_dim(p, 'width') for p in plots]
    heights = [svg_dim(p, 'height') for p in plots]
    xshifts = numpy.cumsum(widths) - numpy.array(widths)
    xshifts = numpy.clip(xshifts - 1, 0, None)  # reduce by one to avoid gaps
    figure = os.path.basename(neutsvgfile).replace('_neut.svg', '')
    letters = [figure_config['figures'][figure]['logo_panel_label'],
               figure_config['figures'][figure]['neut_panel_label']]
    fig_elements = []
    for letter, p, x in zip(letters, plots, xshifts):
        fig_elements.append(svgutils.compose.SVG(p).move(x, 0))
        fig_elements.append(svgutils.compose.Text(letter, 15, 25, weight='bold',
                                                  size='26', font='Arial').move(x, 0))
    f = svgutils.compose.Figure(sum(widths), max(heights), *fig_elements)
    f.save(svgfile)
    pdffile = svg_to_pdf(svgfile)
    print(f"\nWriting figure to {svgfile} and {pdffile}")
    showPDF(pdffile)
```

    
    Writing figure to results/figures/2009_age_53_samples_logo_and_neut.svg and results/figures/2009_age_53_samples_logo_and_neut.pdf



![png](analyze_neut_files/analyze_neut_45_1.png)


    
    Writing figure to results/figures/Hensley_sera_logo_and_neut.svg and results/figures/Hensley_sera_logo_and_neut.pdf



![png](analyze_neut_files/analyze_neut_45_3.png)


    
    Writing figure to results/figures/VIDD_sera_logo_and_neut.svg and results/figures/VIDD_sera_logo_and_neut.pdf



![png](analyze_neut_files/analyze_neut_45_5.png)


    
    Writing figure to results/figures/antibody_lower_head_logo_and_neut.svg and results/figures/antibody_lower_head_logo_and_neut.pdf



![png](analyze_neut_files/analyze_neut_45_7.png)


    
    Writing figure to results/figures/antibody_region_B_logo_and_neut.svg and results/figures/antibody_region_B_logo_and_neut.pdf



![png](analyze_neut_files/analyze_neut_45_9.png)


    
    Writing figure to results/figures/antibody_spikein_logo_and_neut.svg and results/figures/antibody_spikein_logo_and_neut.pdf



![png](analyze_neut_files/analyze_neut_45_11.png)


    
    Writing figure to results/figures/ferret_logo_and_neut.svg and results/figures/ferret_logo_and_neut.pdf



![png](analyze_neut_files/analyze_neut_45_13.png)


Now tabulate the fit parameters for the serum / viruses that we plotted in the figures:


```python
fit_table = (
    fits.fitParams()
    .query('replicate == "average"')
    .merge(pd.DataFrame.from_records(sera_viruses_plotted, columns=['serum', 'virus']),
           how='inner')
    [['serum', 'virus', 'ic50_str', 'slope', 'top', 'bottom']]
    .drop_duplicates()
    .sort_values(['serum', 'virus'], ascending=[True, False])
    )

display(HTML(fit_table.to_html(index=False)))

fit_table_file = os.path.join(outdir, 'neut_assay_figs_fit_params.csv')
print(f"Writing table to {fit_table_file}")
fit_table.to_csv(fit_table_file, index=False, float_format='%.3g')
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>serum</th>
      <th>virus</th>
      <th>ic50_str</th>
      <th>slope</th>
      <th>top</th>
      <th>bottom</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2009-age-53</td>
      <td>wt</td>
      <td>0.00124</td>
      <td>1.779855</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>syn</td>
      <td>0.00153</td>
      <td>2.317307</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>L157D</td>
      <td>0.00481</td>
      <td>1.976588</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>K189D</td>
      <td>0.000548</td>
      <td>1.858524</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>K160T</td>
      <td>0.00338</td>
      <td>2.118436</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>F193D</td>
      <td>0.00234</td>
      <td>1.856294</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53</td>
      <td>F159G</td>
      <td>0.00122</td>
      <td>1.937589</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>wt</td>
      <td>0.00172</td>
      <td>1.208404</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>L157D</td>
      <td>0.00931</td>
      <td>2.644220</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>K160T</td>
      <td>0.00387</td>
      <td>2.206106</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-53-plus-2-months</td>
      <td>F193D</td>
      <td>0.00418</td>
      <td>1.843438</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>wt</td>
      <td>0.000318</td>
      <td>2.252406</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>syn</td>
      <td>0.000386</td>
      <td>1.669909</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>L157D</td>
      <td>0.000213</td>
      <td>1.780584</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>K189D</td>
      <td>0.000258</td>
      <td>1.550414</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>K160T</td>
      <td>0.000649</td>
      <td>1.976921</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>F193D</td>
      <td>0.000386</td>
      <td>2.065217</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-64</td>
      <td>F159G</td>
      <td>0.0108</td>
      <td>3.297095</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>wt</td>
      <td>0.000328</td>
      <td>2.008450</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>syn</td>
      <td>0.000412</td>
      <td>2.109484</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>N121E</td>
      <td>0.000332</td>
      <td>1.993601</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>L157D</td>
      <td>0.00027</td>
      <td>1.996908</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>K189D</td>
      <td>0.000202</td>
      <td>1.603608</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>K160T</td>
      <td>0.00145</td>
      <td>2.568102</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>F193D</td>
      <td>&gt;0.0109</td>
      <td>21.576706</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2009-age-65</td>
      <td>F159G</td>
      <td>0.000805</td>
      <td>1.974978</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>wt</td>
      <td>0.00118</td>
      <td>1.989081</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>syn</td>
      <td>0.00164</td>
      <td>2.586569</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>L157D</td>
      <td>0.00107</td>
      <td>2.076028</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>K189D</td>
      <td>0.0018</td>
      <td>2.127158</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>K160T</td>
      <td>0.00329</td>
      <td>2.515296</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>F193D</td>
      <td>&gt;0.014</td>
      <td>10.432859</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2010-age-21</td>
      <td>F159G</td>
      <td>0.00148</td>
      <td>2.362529</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>wt</td>
      <td>0.00126</td>
      <td>2.430043</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>R220D</td>
      <td>&gt;0.0102</td>
      <td>23.974538</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>K189D</td>
      <td>0.000868</td>
      <td>1.782274</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-prevacc</td>
      <td>F159G</td>
      <td>&gt;0.0102</td>
      <td>9.591964</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>wt</td>
      <td>8.54e-05</td>
      <td>2.609425</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>R220D</td>
      <td>0.000141</td>
      <td>3.151577</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>K189D</td>
      <td>0.000132</td>
      <td>1.937381</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-25-vacc</td>
      <td>F159G</td>
      <td>&gt;0.00117</td>
      <td>1.170513</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-29-prevacc</td>
      <td>wt</td>
      <td>0.00637</td>
      <td>2.061859</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-29-prevacc</td>
      <td>K144E</td>
      <td>0.00561</td>
      <td>1.921607</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-29-prevacc</td>
      <td>F159G</td>
      <td>0.00954</td>
      <td>3.335905</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-29-vacc</td>
      <td>wt</td>
      <td>0.000299</td>
      <td>3.208098</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-29-vacc</td>
      <td>K144E</td>
      <td>0.000559</td>
      <td>3.473683</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-29-vacc</td>
      <td>F159G</td>
      <td>0.000293</td>
      <td>2.981604</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-48-prevacc</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
      <td>2.134402</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>wt</td>
      <td>8.33e-05</td>
      <td>1.779256</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>K189D</td>
      <td>0.000413</td>
      <td>1.311547</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>K144E</td>
      <td>7.23e-05</td>
      <td>2.387690</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-48-vacc</td>
      <td>F159G</td>
      <td>0.000149</td>
      <td>1.457277</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-49-prevacc</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
      <td>0.303021</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2015-age-49-vacc</td>
      <td>wt</td>
      <td>0.00343</td>
      <td>1.773101</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-1C04</td>
      <td>wt</td>
      <td>0.0868</td>
      <td>1.439844</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-1C04</td>
      <td>K82A</td>
      <td>&gt;5</td>
      <td>9.513325</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-4C01</td>
      <td>wt</td>
      <td>0.0207</td>
      <td>2.773914</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-4C01</td>
      <td>K160T</td>
      <td>0.0202</td>
      <td>2.271633</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-4C01</td>
      <td>F193D</td>
      <td>&gt;0.5</td>
      <td>10.334572</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>wt</td>
      <td>0.362</td>
      <td>1.502185</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>N121E</td>
      <td>&gt;3</td>
      <td>9.580069</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>F193D</td>
      <td>0.13</td>
      <td>1.819803</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-4F03</td>
      <td>F159G</td>
      <td>0.0649</td>
      <td>1.831487</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-5A01</td>
      <td>wt</td>
      <td>0.0884</td>
      <td>2.103692</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-5A01</td>
      <td>K160T</td>
      <td>0.549</td>
      <td>1.852408</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>antibody-5A01</td>
      <td>F159G</td>
      <td>&gt;1</td>
      <td>1.384432</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>wt</td>
      <td>9.84e-05</td>
      <td>2.659119</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>K189D</td>
      <td>0.000292</td>
      <td>2.014105</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>K144E</td>
      <td>0.000176</td>
      <td>3.166316</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>F193D</td>
      <td>0.000484</td>
      <td>3.940897</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-postinf</td>
      <td>F159G</td>
      <td>0.000186</td>
      <td>2.015143</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-1-preinf</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
      <td>8.567956</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>wt</td>
      <td>0.000343</td>
      <td>2.671063</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>L157D</td>
      <td>0.000486</td>
      <td>2.964182</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>K189D</td>
      <td>0.000724</td>
      <td>2.611096</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>K144E</td>
      <td>0.000636</td>
      <td>2.802963</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>F193D</td>
      <td>0.0012</td>
      <td>3.102091</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-postinf</td>
      <td>F159G</td>
      <td>0.000341</td>
      <td>2.252597</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-2-preinf</td>
      <td>wt</td>
      <td>&gt;0.00206</td>
      <td>9.757383</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>wt</td>
      <td>0.000375</td>
      <td>2.435213</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>L157D</td>
      <td>0.000265</td>
      <td>1.905240</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>K189D</td>
      <td>0.00113</td>
      <td>1.981848</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>K160T</td>
      <td>0.000471</td>
      <td>2.376600</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-postinf</td>
      <td>F193D</td>
      <td>0.00505</td>
      <td>3.129703</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-Pitt-3-preinf</td>
      <td>wt</td>
      <td>&gt;0.00617</td>
      <td>8.589477</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-WHO-Perth2009</td>
      <td>wt</td>
      <td>0.00476</td>
      <td>2.111494</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>wt</td>
      <td>0.00192</td>
      <td>2.256116</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>K189D</td>
      <td>0.0115</td>
      <td>3.008475</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>K160T</td>
      <td>0.00871</td>
      <td>2.421736</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>F193D</td>
      <td>&gt;0.0125</td>
      <td>2.561249</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>ferret-WHO-Victoria2011</td>
      <td>F159G</td>
      <td>0.00601</td>
      <td>2.520402</td>
      <td>1</td>
      <td>0</td>
    </tr>
  </tbody>
</table>


    Writing table to results/neutralization_assays/neut_assay_figs_fit_params.csv



```python

```
