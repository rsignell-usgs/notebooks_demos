---
layout: notebook
title: ""
---


# Exploring the THREDDS catalog with Unidata's Siphon

[Siphon](http://siphon.readthedocs.io/en/latest/) is a Python module for accessing data hosted on a THREDDS data server.
Siphon works by parsing the catalog XML and exposing it with higher level functions.

In this notebook we will explore data available on the Central & Northern California Ocean Observing System (CeNCOOS) THREDDS. The cell below extracts the catalog information

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
from siphon.catalog import TDSCatalog

catalog = TDSCatalog('http://thredds.cencoos.org/thredds/catalog.xml')


info = """
Catalog information
-------------------

Base THREDDS URL: {}
Catalog name: {}
Catalog URL: {}
Metadata: {}
""".format(catalog.base_tds_url,
           catalog.catalog_name,
           catalog.catalog_url,
           catalog.metadata)

print(info)
```
<div class="output_area"><div class="prompt"></div>
<pre>
    
    Catalog information
    -------------------
    
    Base THREDDS URL: http://thredds.cencoos.org
    Catalog name: CeNCOOS
    Catalog URL: http://thredds.cencoos.org/thredds/catalog.xml
    Metadata: {}
    

</pre>
</div>
Unfortunately this catalog has no metadata. So let's check what kind of services are available.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
for service in catalog.services:
    print(service.name)
```
<div class="output_area"><div class="prompt"></div>
<pre>
    all
    allandsos
    wms

</pre>
</div>
And what datasets are there?

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
print('\n'.join(catalog.datasets.keys()))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    California Coastal Regional Ocean Modeling System (ROMS) Nowcast
    California Coastal Regional Ocean Modeling System (ROMS) Forecast
    Monterey Bay (MB) Regional Ocean Modeling System (ROMS) Nowcast
    Monterey Bay (MB) Regional Ocean Modeling System (ROMS) Forecast
    Southern California Bight (SCB) Regional Ocean Modeling System (ROMS) Nowcast
    UCSC California Current System Model
    HAB Cellular Domoic Acid Forecast
    HAB Cellular Domoic Acid Nowcast
    HAB Particulate Domoic Acid Forecast
    HAB Particulate Domoic Acid Nowcast
    HAB Pseudo Nitzschia Forecast
    HAB Pseudo Nitzschia Nowcast
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Relative Humidity
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Total Precipitation
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Visibility
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Wind 10 m
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Air Temp 2 m
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Cloud Base
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Ground and Sea Temp
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Pressure Reduce to MSL
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Net Short-wave Radiation Surface
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) IR Heat Flux
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Monthly Averaged Winds
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Wind 10 m (Historical)
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Ground and Sea Temp (Historical)
    Coupled Ocean/Atmosphere Mesoscale Prediction System (COAMPS) Air Temp 2 m (Historical)
    Hybrid Coordinate Ocean Model
    Southern California Regional NCOM Model
    Maurer Meteorological Data
    Global 1-km Sea Surface Temperature (G1SST)
    High Resolution Chlorophyll-a concentration from MODIS/Aqua (1 Day Composite)
    High Resolution Chlorophyll-a concentration from MODIS/Aqua (8 Day Composite)
    High Resolution Chlorophyll-a concentration from MODIS/Aqua (1 Month Composite)
    High Resolution Sea Surface Temperature from the Advanced Very-High Resolution Radiometer (1 Day Composite)
    High Resolution Sea Surface Temperature from the Advanced Very-High Resolution Radiometer (8 Day Composite)
    High Resolution Sea Surface Temperature from the Advanced Very-High Resolution Radiometer (1 Month Composite)
    Ocean Surface Currents Monthly Averaged - CORDC High-Frequency Radar (US West Coast), 6 km

</pre>
</div>
It looks like model runs as well as satellite and HFR data. One can also check the catalog refs for more information

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
print('\n'.join(catalog.catalog_refs.keys()))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Global
    Dynamic
    Static
    HF RADAR, US West Coast
    HF RADAR, US West Coast (GNOME Format)

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
ref = catalog.catalog_refs['Global']

[value for value in dir(ref) if not value.startswith('__')]
```




    ['follow', 'href', 'name', 'title']



<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
info = """
Href: {}
Name: {}
Title: {}
""".format(
    ref.href,
    ref.name,
    ref.title)

print(info)
```
<div class="output_area"><div class="prompt"></div>
<pre>
    
    Href: http://thredds.cencoos.org/thredds/global.xml
    Name: 
    Title: Global
    

</pre>
</div>
The `follow` method navigates to that catalog `ref` and returns a new `siphon.catalog.TDSCatalog` object for that part of the THREDDS catalog.

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
cat = ref.follow()

print(type(cat))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    <class 'siphon.catalog.TDSCatalog'>

</pre>
</div>
That makes it easier to explore a small subset of the datasets available in the catalog.
Here are the data from the *Global* subset.

<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
print('\n'.join(cat.datasets.keys()))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    NCEP Reanalysis Daily Averages Surface Flux
    Global 1-km Sea Surface Temperature (G1SST)
    NCEP Global Forecast System Model (GFS)
    Aquarius V 3.0 Scatterometer Daily Aggregate
    Aquarius V 3.0 Scatterometer Seven-Day Aggregate
    Aquarius V 3.0 Scatterometer Monthly Aggregate
    Aquarius V 3.0 Radiometer Daily Aggregate
    Aquarius V 3.0 Radiometer Seven-Day Aggregate
    Aquarius V 3.0 Radiometer Monthly Aggregate
    Aquarius V 4.0 Scatterometer Daily Aggregate
    Aquarius V 4.0 Scatterometer Seven-Day Aggregate
    Aquarius V 4.0 Scatterometer Monthly Aggregate
    Aquarius V 4.0 Radiometer Daily Aggregate
    Aquarius V 4.0 Radiometer Seven-Day Aggregate
    Aquarius V 4.0 Radiometer Monthly Aggregate

</pre>
</div>
Let's extract the `Global 1-km Sea Surface Temperature` dataset from the global `ref`.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
dataset = 'Global 1-km Sea Surface Temperature (G1SST)'

ds = cat.datasets[dataset]

ds.name, ds.url_path
```




    ('Global 1-km Sea Surface Temperature (G1SST)', 'G1_SST_GLOBAL.nc')



Siphon has a `ncss` (NetCDF subset service) access, here is a quote from the documentation:

> This module contains code to support making data requests to
the NetCDF subset service (NCSS) on a THREDDS Data Server (TDS). This includes
forming proper queries as well as parsing the returned data.

Let's check if the catalog offers the `NetcdfSubset` in the `access_urls`.

<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
for name, ds in catalog.datasets.items():
    if ds.access_urls:
        print(name)
```

All `access_urls` returned empty.... Maybe that is just a metadata issue because there is `NetcdfSubset` access when navigating in the webpage.

<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
from IPython.display import IFrame

url = 'http://thredds.cencoos.org/thredds/catalog.html?dataset=G1_SST_US_WEST_COAST'

IFrame(url, width=800, height=550)
```





        <iframe
            width="800"
            height="550"
            src="http://thredds.cencoos.org/thredds/catalog.html?dataset=G1_SST_US_WEST_COAST"
            frameborder="0"
            allowfullscreen
        ></iframe>
        



To finish the post let's check if there is any WMS service available and overlay the data in a slippy (interactive) map.

<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
services = [service for service in catalog.services if service.name == 'wms']

services
```




    [<siphon.catalog.SimpleService at 0x7fbad99427b8>]



Found only one, let's tease that out and check the URL.

<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
service = services[0]

url = service.base

url
```




    'http://pdx.axiomalaska.com/ncWMS/wms'



OWSLib helps to inspect the available layers before plotting. Here we will get the first layer that has G1_SST_US_WEST_COAST on it.

Note, however, we are skipping the discovery step of the `wms` information and hard-coding it instead.
That is to save time because parsing the URL [http://pdx.axiomalaska.com/ncWMS/wms](http://pdx.axiomalaska.com/ncWMS/wms) takes ~ 10 minutes. See [this](https://github.com/ioos/notebooks_demos/pull/171#issuecomment-271705056) issue for more information.

<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
if False:
    from owslib.wms import WebMapService
    web_map_services = WebMapService(url)
    layer = [key for key in web_map_services.contents.keys() if 'G1_SST_US_WEST_COAST' in key][0]
    wms = web_map_services.contents[layer]

    title = wms.title
    lon = (wms.boundingBox[0] + wms.boundingBox[2]) / 2.
    lat = (wms.boundingBox[1] + wms.boundingBox[3]) / 2.
    time = wms.defaulttimeposition
else:
    layer = 'G1_SST_US_WEST_COAST/analysed_sst'
    title = 'Sea Surface Temperature'
    lon, lat = -122.50, 39.50
    time = 'undefined'
```

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
import folium

m = folium.Map(location=[lat, lon], zoom_start=4)

folium.WmsTileLayer(
    name='{} at {}'.format(title, time),
    url=url,
    layers=layer,
    format='image/png'
).add_to(m)

folium.LayerControl().add_to(m)

m
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;base64,CiAgICAgICAgPCFET0NUWVBFIGh0bWw+CiAgICAgICAgPGhlYWQ+CiAgICAgICAgICAgIAogICAgICAgIAogICAgICAgICAgICA8bWV0YSBodHRwLWVxdWl2PSJjb250ZW50LXR5cGUiIGNvbnRlbnQ9InRleHQvaHRtbDsgY2hhcnNldD1VVEYtOCIgLz4KICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0LzAuNy4zL2xlYWZsZXQuanMiPjwvc2NyaXB0PgogICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vYWpheC5nb29nbGVhcGlzLmNvbS9hamF4L2xpYnMvanF1ZXJ5LzEuMTEuMS9qcXVlcnkubWluLmpzIj48L3NjcmlwdD4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxzY3JpcHQgc3JjPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9qcy9ib290c3RyYXAubWluLmpzIj48L3NjcmlwdD4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9MZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy8yLjAuMi9sZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy5taW4uanMiPjwvc2NyaXB0PgogICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8wLjQuMC9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXItc3JjLmpzIj48L3NjcmlwdD4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMC40LjAvbGVhZmxldC5tYXJrZXJjbHVzdGVyLmpzIj48L3NjcmlwdD4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvbGVhZmxldC8wLjcuMy9sZWFmbGV0LmNzcyIgLz4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC5taW4uY3NzIiAvPgogICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiIC8+CiAgICAgICAgCiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIAogICAgICAgIAogICAgICAgICAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuMS4wL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIgLz4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuY3NzIiAvPgogICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMC40LjAvTWFya2VyQ2x1c3Rlci5EZWZhdWx0LmNzcyIgLz4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvbGVhZmxldC5tYXJrZXJjbHVzdGVyLzAuNC4wL01hcmtlckNsdXN0ZXIuY3NzIiAvPgogICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhdy5naXRodWJ1c2VyY29udGVudC5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIiAvPgogICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAgICAgPHN0eWxlPgoKICAgICAgICAgICAgaHRtbCwgYm9keSB7CiAgICAgICAgICAgICAgICB3aWR0aDogMTAwJTsKICAgICAgICAgICAgICAgIGhlaWdodDogMTAwJTsKICAgICAgICAgICAgICAgIG1hcmdpbjogMDsKICAgICAgICAgICAgICAgIHBhZGRpbmc6IDA7CiAgICAgICAgICAgICAgICB9CgogICAgICAgICAgICAjbWFwIHsKICAgICAgICAgICAgICAgIHBvc2l0aW9uOmFic29sdXRlOwogICAgICAgICAgICAgICAgdG9wOjA7CiAgICAgICAgICAgICAgICBib3R0b206MDsKICAgICAgICAgICAgICAgIHJpZ2h0OjA7CiAgICAgICAgICAgICAgICBsZWZ0OjA7CiAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgIDwvc3R5bGU+CiAgICAgICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAgICAgPHN0eWxlPiAjbWFwXzcyMDUxMTJlMDk1MTQyODFhM2U2ZGRmMmE2NzZlZTAyIHsKICAgICAgICAgICAgICAgIHBvc2l0aW9uIDogcmVsYXRpdmU7CiAgICAgICAgICAgICAgICB3aWR0aCA6IDEwMC4wJTsKICAgICAgICAgICAgICAgIGhlaWdodDogMTAwLjAlOwogICAgICAgICAgICAgICAgbGVmdDogMC4wJTsKICAgICAgICAgICAgICAgIHRvcDogMC4wJTsKICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgPC9zdHlsZT4KICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICA8L2hlYWQ+CiAgICAgICAgPGJvZHk+CiAgICAgICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAgICAgPGRpdiBjbGFzcz0iZm9saXVtLW1hcCIgaWQ9Im1hcF83MjA1MTEyZTA5NTE0MjgxYTNlNmRkZjJhNjc2ZWUwMiIgPjwvZGl2PgogICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgIDwvYm9keT4KICAgICAgICA8c2NyaXB0PgogICAgICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCgogICAgICAgICAgICB2YXIgc291dGhXZXN0ID0gTC5sYXRMbmcoLTkwLCAtMTgwKTsKICAgICAgICAgICAgdmFyIG5vcnRoRWFzdCA9IEwubGF0TG5nKDkwLCAxODApOwogICAgICAgICAgICB2YXIgYm91bmRzID0gTC5sYXRMbmdCb3VuZHMoc291dGhXZXN0LCBub3J0aEVhc3QpOwoKICAgICAgICAgICAgdmFyIG1hcF83MjA1MTEyZTA5NTE0MjgxYTNlNmRkZjJhNjc2ZWUwMiA9IEwubWFwKCdtYXBfNzIwNTExMmUwOTUxNDI4MWEzZTZkZGYyYTY3NmVlMDInLCB7CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBjZW50ZXI6WzM5LjUsLTEyMi41XSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIHpvb206IDQsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBtYXhCb3VuZHM6IGJvdW5kcywKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGxheWVyczogW10sCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBjcnM6IEwuQ1JTLkVQU0czODU3CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgIAogICAgICAgIAogICAgICAgIAogICAgICAgICAgICAKICAgICAgICAgICAgdmFyIHRpbGVfbGF5ZXJfOTcyM2YxYTg0NDEwNDE3YzlkOWZmNTMwMDMzMGFmMWEgPSBMLnRpbGVMYXllcigKICAgICAgICAgICAgICAgICdodHRwczovL3tzfS50aWxlLm9wZW5zdHJlZXRtYXAub3JnL3t6fS97eH0ve3l9LnBuZycsCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgbWF4Wm9vbTogMTgsCiAgICAgICAgICAgICAgICAgICAgbWluWm9vbTogMSwKICAgICAgICAgICAgICAgICAgICBhdHRyaWJ1dGlvbjogJ0RhdGEgYnkgPGEgaHJlZj0iaHR0cDovL29wZW5zdHJlZXRtYXAub3JnIj5PcGVuU3RyZWV0TWFwPC9hPiwgdW5kZXIgPGEgaHJlZj0iaHR0cDovL3d3dy5vcGVuc3RyZWV0bWFwLm9yZy9jb3B5cmlnaHQiPk9EYkw8L2E+LicsCiAgICAgICAgICAgICAgICAgICAgZGV0ZWN0UmV0aW5hOiBmYWxzZQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwXzcyMDUxMTJlMDk1MTQyODFhM2U2ZGRmMmE2NzZlZTAyKTsKCiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAgIAogICAgICAgICAgICB2YXIgbWFjcm9fZWxlbWVudF84NDNkMWJhYWFkNGI0ZDAzYTY0MjRhZjU1MDcxNThhNyA9IEwudGlsZUxheWVyLndtcygKICAgICAgICAgICAgICAgICdodHRwOi8vcGR4LmF4aW9tYWxhc2thLmNvbS9uY1dNUy93bXMnLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGxheWVyczogJ0cxX1NTVF9VU19XRVNUX0NPQVNUL2FuYWx5c2VkX3NzdCcsCiAgICAgICAgICAgICAgICAgICAgc3R5bGVzOiAnJywKICAgICAgICAgICAgICAgICAgICBmb3JtYXQ6ICdpbWFnZS9wbmcnLAogICAgICAgICAgICAgICAgICAgIHRyYW5zcGFyZW50OiB0cnVlLAogICAgICAgICAgICAgICAgICAgIHZlcnNpb246ICcxLjEuMScsCiAgICAgICAgICAgICAgICAgICAgCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfNzIwNTExMmUwOTUxNDI4MWEzZTZkZGYyYTY3NmVlMDIpOwoKICAgICAgICAKICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgICAgIHZhciBsYXllcl9jb250cm9sXzgwMDMyM2Q5NTNjNTQ3MDBhODc5N2E0ZGFlMWJlNGRlID0gewogICAgICAgICAgICAgICAgYmFzZV9sYXllcnMgOiB7ICJvcGVuc3RyZWV0bWFwIiA6IHRpbGVfbGF5ZXJfOTcyM2YxYTg0NDEwNDE3YzlkOWZmNTMwMDMzMGFmMWEsIH0sCiAgICAgICAgICAgICAgICBvdmVybGF5cyA6IHsgIlNlYSBTdXJmYWNlIFRlbXBlcmF0dXJlIGF0IHVuZGVmaW5lZCIgOiBtYWNyb19lbGVtZW50Xzg0M2QxYmFhYWQ0YjRkMDNhNjQyNGFmNTUwNzE1OGE3LCB9CiAgICAgICAgICAgICAgICB9OwogICAgICAgICAgICBMLmNvbnRyb2wubGF5ZXJzKAogICAgICAgICAgICAgICAgbGF5ZXJfY29udHJvbF84MDAzMjNkOTUzYzU0NzAwYTg3OTdhNGRhZTFiZTRkZS5iYXNlX2xheWVycywKICAgICAgICAgICAgICAgIGxheWVyX2NvbnRyb2xfODAwMzIzZDk1M2M1NDcwMGE4Nzk3YTRkYWUxYmU0ZGUub3ZlcmxheXMKICAgICAgICAgICAgICAgICkuYWRkVG8obWFwXzcyMDUxMTJlMDk1MTQyODFhM2U2ZGRmMmE2NzZlZTAyKTsKICAgICAgICAKICAgICAgICAKICAgICAgICAKICAgICAgICA8L3NjcmlwdD4KICAgICAgICA=" style="position:absolute;width:100%;height:100%;left:0;top:0;"></iframe></div></div>



<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2017-01-18-siphon-explore-thredds.ipynb)
this notebook, or see a static view [here](http://nbviewer.ipython.org/urls/raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2017-01-18-siphon-explore-thredds.ipynb).
