---
layout: notebook
title: ""
---


# The Boston Light Swim temperature analysis with Python

In the past we demonstrated how to perform a CSW catalog search with [`OWSLib`](https://ioos.github.io/notebooks_demos//notebooks/2016-12-19-exploring_csw),
and how to obtain near real-time data with [`pyoos`](https://ioos.github.io/notebooks_demos//notebooks/2016-10-12-fetching_data).
In this notebook we will use both to find all observations and model data around the Boston Harbor to access the sea water temperature.


This workflow is part of an example to advise swimmers of the annual [Boston lighthouse swim](http://bostonlightswim.org/) of the Boston Harbor water temperature conditions prior to the race. For more information regarding the workflow presented here see [Signell, Richard P.; Fernandes, Filipe; Wilcox, Kyle.   2016. "Dynamic Reusable Workflows for Ocean Science." *J. Mar. Sci. Eng.* 4, no. 4: 68](http://dx.doi.org/10.3390/jmse4040068).

(This notebook uses a custom `ioos_tools` module that needs to be added to the path separately. We recommend cloning the [repository](https://github.com/ioos/notebooks_demos) on GitHub which already includes the most update version of `ioos_tools`.)

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
import os
import sys
import warnings

ioos_tools = os.path.join(os.path.pardir)
sys.path.append(ioos_tools)

# Suppresing warnings for a "pretty output."
warnings.simplefilter("ignore")
```

This notebook is quite big and complex,
so to help us keep things organized we'll define a cell with the most important options and switches.

Below we can define the date,
bounding box, phenomena `SOS` and `CF` names and units,
and the catalogs we will search.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
%%writefile config.yaml

# Specify a YYYY-MM-DD hh:mm:ss date or integer day offset.
# If both start and stop are offsets they will be computed relative to datetime.today() at midnight.
# Use the dates commented below to reproduce the last Boston Light Swim event forecast.
date:
    start: -5 # 2016-8-16 00:00:00
    stop: +4 # 2016-8-29 00:00:00

run_name: 'latest'

# Boston harbor.
region:
    bbox: [-71.3, 42.03, -70.57, 42.63]
    crs: 'urn:ogc:def:crs:OGC:1.3:CRS84'

sos_name: 'sea_water_temperature'

cf_names:
    - sea_water_temperature
    - sea_surface_temperature
    - sea_water_potential_temperature
    - equivalent_potential_temperature
    - sea_water_conservative_temperature
    - pseudo_equivalent_potential_temperature

units: 'celsius'

catalogs:
    - https://data.ioos.us/csw
    - http://gamone.whoi.edu/csw
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Overwriting config.yaml

</pre>
</div>
We'll print some of the search configuration options along the way to keep track of them.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
import shutil
from datetime import datetime
from ioos_tools.ioos import parse_config

config = parse_config('config.yaml')

# Saves downloaded data into a temporary directory.
save_dir = os.path.abspath(config['run_name'])
if os.path.exists(save_dir):
    shutil.rmtree(save_dir)
os.makedirs(save_dir)

fmt = '{:*^64}'.format
print(fmt('Saving data inside directory {}'.format(save_dir)))
print(fmt(' Run information '))
print('Run date: {:%Y-%m-%d %H:%M:%S}'.format(datetime.utcnow()))
print('Start: {:%Y-%m-%d %H:%M:%S}'.format(config['date']['start']))
print('Stop: {:%Y-%m-%d %H:%M:%S}'.format(config['date']['stop']))
print('Bounding box: {0:3.2f}, {1:3.2f},'
      '{2:3.2f}, {3:3.2f}'.format(*config['region']['bbox']))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Saving data inside directory /home/filipe/IOOS/notebooks_demos/notebooks/latest
    *********************** Run information ************************
    Run date: 2017-04-20 20:04:17
    Start: 2017-04-15 00:00:00
    Stop: 2017-04-24 00:00:00
    Bounding box: -71.30, 42.03,-70.57, 42.63

</pre>
</div>
We already created an `OWSLib.fes` filter [before](https://ioos.github.io/notebooks_demos//notebooks/2016-12-19-exploring_csw).
The main difference here is that we do not want the atmosphere model data,
so we are filtering out all the `GRIB-2` data format.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
def make_filter(config):
    from owslib import fes
    from ioos_tools.ioos import fes_date_filter
    kw = dict(wildCard='*', escapeChar='\\',
              singleChar='?', propertyname='apiso:AnyText')

    or_filt = fes.Or([fes.PropertyIsLike(literal=('*%s*' % val), **kw)
                      for val in config['cf_names']])

    not_filt = fes.Not([fes.PropertyIsLike(literal='GRIB-2', **kw)])

    begin, end = fes_date_filter(config['date']['start'],
                                 config['date']['stop'])
    bbox_crs = fes.BBox(config['region']['bbox'],
                        crs=config['region']['crs'])
    filter_list = [fes.And([bbox_crs, begin, end, or_filt, not_filt])]
    return filter_list


filter_list = make_filter(config)
```

In the cell below we ask the catalog for all the returns that match the filter and have an OPeNDAP endpoint.

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
from ioos_tools.ioos import service_urls, get_csw_records
from owslib.csw import CatalogueServiceWeb


dap_urls = []
print(fmt(' Catalog information '))
for endpoint in config['catalogs']:
    print("URL: {}".format(endpoint))

    csw = CatalogueServiceWeb(endpoint, timeout=120)
    csw = get_csw_records(csw, filter_list, esn='full')
    OPeNDAP = service_urls(csw.records, identifier='OPeNDAP:OPeNDAP')
    odp = service_urls(csw.records, identifier='urn:x-esri:specification:ServiceType:odp:url')
    dap = OPeNDAP + odp
    dap_urls.extend(dap)

    print("Number of datasets available: {}".format(len(csw.records.keys())))

    for rec, item in csw.records.items():
        print('{}'.format(item.title))
    if dap:
        print(fmt(' DAP '))
        for url in dap:
            print('{}.html'.format(url))
    print('\n')

# Get only unique endpoints.
dap_urls = list(set(dap_urls))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Catalog information **********************
    URL: https://data.ioos.us/csw
    Number of datasets available: 30
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 MET Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 MET Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 OPTICS3m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 OPTICS3m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 OPTODE51m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 OPTODE51m Massachusetts Bay
    NOAA/NCEP Global Forecast System (GFS) Atmospheric Model
    ROMS ESPRESSO Real-Time Operational IS4DVAR Forecast System Version 2 (NEW) 2013-present FMRC Averages
    ROMS ESPRESSO Real-Time Operational IS4DVAR Forecast System Version 2 (NEW) 2013-present FMRC History
    urn:ioos:station:NOAA.NOS.CO-OPS:8443970 station, Boston, MA
    COAWST Modeling System: USEast: ROMS-WRF-SWAN coupled model (aka CNAPS)
    Coupled Northwest Atlantic Prediction System (CNAPS)
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near ASTORIA CANYON, OR from 2016/03/30 17:00:00 to 2017/04/20 00:03:02.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near CLATSOP SPIT, OR from 2016/10/12 17:00:00 to 2017/04/20 00:02:35.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near GRAYS HARBOR, WA from 2016/03/16 22:00:00 to 2017/04/19 23:56:09.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near LOWER COOK INLET, AK from 2016/12/16 00:00:00 to 2017/04/20 00:09:43.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near OCEAN STATION PAPA from 2015/01/01 01:00:00 to 2017/04/20 00:00:57.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near SCRIPPS NEARSHORE, CA from 2015/01/07 23:00:00 to 2017/04/20 00:01:37.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near UMPQUA OFFSHORE, OR from 2016/12/02 21:00:00 to 2017/04/19 23:48:29.
    G1SST, 1km blended SST
    HYbrid Coordinate Ocean Model (HYCOM): Global
    NECOFS (FVCOM) - Scituate - Latest Forecast
    NECOFS GOM3 (FVCOM) - Northeast US - Latest Forecast
    NECOFS Massachusetts (FVCOM) - Boston - Latest Forecast
    NECOFS Massachusetts (FVCOM) - Massachusetts Coastal - Latest Forecast
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 ACCELEROMETER Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 ACCELEROMETER Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 CTD1m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 CTD1m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 CTD20m Massachusetts Bay
    ***************************** DAP ******************************
    http://oos.soest.hawaii.edu/thredds/dodsC/hioos/model/atm/ncep_global/NCEP_Global_Atmospheric_Model_best.ncd.html
    http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/avg/ESPRESSO_Real-Time_v2_Averages_Best.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/his/ESPRESSO_Real-Time_v2_History_Best.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/036p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/139p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/162p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/166p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/179p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/201p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/204p1_rt.nc.html
    http://thredds.secoora.org/thredds/dodsC/G1_SST_GLOBAL.nc.html
    http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/Accelerometer/HistoricRealtime/Agg.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/CTD1m/HistoricRealtime/Agg.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/Met/HistoricRealtime/Agg.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/OPTICS_S3m/HistoricRealtime/Agg.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/OPTODE51m/HistoricRealtime/Agg.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/Accelerometer/Realtime.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/CTD1m/Realtime.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/CTD20m/Realtime.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/Met/Realtime.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/OPTICS_S3m/Realtime.ncml.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/OPTODE51m/Realtime.ncml.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_GOM3_FORECAST.nc.html
    
    
    URL: http://gamone.whoi.edu/csw
    Number of datasets available: 0
    
    

</pre>
</div>
We found some models, and observations from NERACOOS there.
However, we do know that there are some buoys from NDBC and CO-OPS available too.
Also, those NERACOOS observations seem to be from a [CTD](http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/CTD1m/HistoricRealtime/Agg.ncml.html) mounted at 65 meters below the sea surface. Rendering them useless from our purpose.

So let's use the catalog only for the models by filtering the observations with `is_station` below.
And we'll rely `CO-OPS` and `NDBC` services for the observations.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
from ioos_tools.ioos import is_station

# Filter out some station endpoints.
non_stations = []
for url in dap_urls:
    try:
        if not is_station(url):
            non_stations.append(url)
    except (RuntimeError, OSError, IOError) as e:
        print("Could not access URL {}. {!r}".format(url, e))

dap_urls = non_stations

print(fmt(' Filtered DAP '))
for url in dap_urls:
    print('{}.html'.format(url))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ************************* Filtered DAP *************************
    http://oos.soest.hawaii.edu/thredds/dodsC/hioos/model/atm/ncep_global/NCEP_Global_Atmospheric_Model_best.ncd.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_GOM3_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc.html
    http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/avg/ESPRESSO_Real-Time_v2_Averages_Best.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/his/ESPRESSO_Real-Time_v2_History_Best.html
    http://thredds.secoora.org/thredds/dodsC/G1_SST_GLOBAL.nc.html
    http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global.html

</pre>
</div>
Now we can use `pyoos` collectors for `NdbcSos`,

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
from pyoos.collectors.ndbc.ndbc_sos import NdbcSos

collector_ndbc = NdbcSos()

collector_ndbc.set_bbox(config['region']['bbox'])
collector_ndbc.end_time = config['date']['stop']
collector_ndbc.start_time = config['date']['start']
collector_ndbc.variables = [config['sos_name']]

ofrs = collector_ndbc.server.offerings
title = collector_ndbc.server.identification.title
print(fmt(' NDBC Collector offerings '))
print('{}: {} offerings'.format(title, len(ofrs)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ******************* NDBC Collector offerings *******************
    National Data Buoy Center SOS: 985 offerings

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
import pandas as pd
from ioos_tools.ioos import collector2table

ndbc = collector2table(collector=collector_ndbc,
                       config=config,
                       col='sea_water_temperature (C)')

if ndbc:
    data = dict(
        station_name=[s._metadata.get('station_name') for s in ndbc],
        station_code=[s._metadata.get('station_code') for s in ndbc],
        sensor=[s._metadata.get('sensor') for s in ndbc],
        lon=[s._metadata.get('lon') for s in ndbc],
        lat=[s._metadata.get('lat') for s in ndbc],
        depth=[s._metadata.get('depth') for s in ndbc],
    )

table = pd.DataFrame(data).set_index('station_code')
table
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>depth</th>
      <th>lat</th>
      <th>lon</th>
      <th>sensor</th>
      <th>station_name</th>
    </tr>
    <tr>
      <th>station_code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>44013</th>
      <td>0.6</td>
      <td>42.346</td>
      <td>-70.651</td>
      <td>urn:ioos:sensor:wmo:44013::watertemp1</td>
      <td>BOSTON 16 NM East of Boston, MA</td>
    </tr>
  </tbody>
</table>
</div>



and `CoopsSos`.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
from pyoos.collectors.coops.coops_sos import CoopsSos

collector_coops = CoopsSos()

collector_coops.set_bbox(config['region']['bbox'])
collector_coops.end_time = config['date']['stop']
collector_coops.start_time = config['date']['start']
collector_coops.variables = [config['sos_name']]

ofrs = collector_coops.server.offerings
title = collector_coops.server.identification.title
print(fmt(' Collector offerings '))
print('{}: {} offerings'.format(title, len(ofrs)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Collector offerings **********************
    NOAA.NOS.CO-OPS SOS: 1145 offerings

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
coops = collector2table(collector=collector_coops,
                        config=config,
                        col='sea_water_temperature (C)')

if coops:
    data = dict(
        station_name=[s._metadata.get('station_name') for s in coops],
        station_code=[s._metadata.get('station_code') for s in coops],
        sensor=[s._metadata.get('sensor') for s in coops],
        lon=[s._metadata.get('lon') for s in coops],
        lat=[s._metadata.get('lat') for s in coops],
        depth=[s._metadata.get('depth') for s in coops],
    )

table = pd.DataFrame(data).set_index('station_code')
table
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>depth</th>
      <th>lat</th>
      <th>lon</th>
      <th>sensor</th>
      <th>station_name</th>
    </tr>
    <tr>
      <th>station_code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>8443970</th>
      <td>None</td>
      <td>42.3548</td>
      <td>-71.0534</td>
      <td>urn:ioos:sensor:NOAA.NOS.CO-OPS:8443970:E1</td>
      <td>Boston, MA</td>
    </tr>
  </tbody>
</table>
</div>



We will join all the observations into an uniform series, interpolated to 1-hour interval, for the model-data comparison.

This step is necessary because the observations can be 7 or 10 minutes resolution,
while the models can be 30 to 60 minutes.

<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
data = ndbc + coops

index = pd.date_range(start=config['date']['start'].replace(tzinfo=None),
                      end=config['date']['stop'].replace(tzinfo=None),
                      freq='1H')

# Preserve metadata with `reindex`.
observations = []
for series in data:
    _metadata = series._metadata
    obs = series.reindex(index=index, limit=1, method='nearest')
    obs._metadata = _metadata
    observations.append(obs)
```

In this next cell we will save the data for quicker access later.

<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
import iris
from ioos_tools.tardis import series2cube

attr = dict(
    featureType='timeSeries',
    Conventions='CF-1.6',
    standard_name_vocabulary='CF-1.6',
    cdm_data_type="Station",
    comment="Data from http://opendap.co-ops.nos.noaa.gov"
)


cubes = iris.cube.CubeList(
    [series2cube(obs, attr=attr) for obs in observations]
)

outfile = os.path.join(save_dir, 'OBS_DATA.nc')
iris.save(cubes, outfile)
```

Now it is time to loop the models we found above,

<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
from iris.exceptions import (CoordinateNotFoundError, ConstraintMismatchError,
                             MergeError)
from ioos_tools.ioos import get_model_name
from ioos_tools.tardis import quick_load_cubes, proc_cube, is_model, get_surface

print(fmt(' Models '))
cubes = dict()
for k, url in enumerate(dap_urls):
    print('\n[Reading url {}/{}]: {}'.format(k+1, len(dap_urls), url))
    try:
        cube = quick_load_cubes(url, config['cf_names'],
                                callback=None, strict=True)
        if is_model(cube):
            cube = proc_cube(cube,
                             bbox=config['region']['bbox'],
                             time=(config['date']['start'],
                                   config['date']['stop']),
                             units=config['units'])
        else:
            print("[Not model data]: {}".format(url))
            continue
        cube = get_surface(cube)
        mod_name = get_model_name(url)
        cubes.update({mod_name: cube})
    except (RuntimeError, ValueError,
            ConstraintMismatchError, CoordinateNotFoundError,
            IndexError) as e:
        print('Cannot get cube for: {}\n{}'.format(url, e))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    **************************** Models ****************************
    
    [Reading url 1/10]: http://oos.soest.hawaii.edu/thredds/dodsC/hioos/model/atm/ncep_global/NCEP_Global_Atmospheric_Model_best.ncd
    Cannot get cube for: http://oos.soest.hawaii.edu/thredds/dodsC/hioos/model/atm/ncep_global/NCEP_Global_Atmospheric_Model_best.ncd
    Cannot find ['sea_water_temperature', 'sea_surface_temperature', 'sea_water_potential_temperature', 'equivalent_potential_temperature', 'sea_water_conservative_temperature', 'pseudo_equivalent_potential_temperature'] in http://oos.soest.hawaii.edu/thredds/dodsC/hioos/model/atm/ncep_global/NCEP_Global_Atmospheric_Model_best.ncd.
    
    [Reading url 2/10]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc
    
    [Reading url 3/10]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_GOM3_FORECAST.nc
    
    [Reading url 4/10]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc
    
    [Reading url 5/10]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc
    
    [Reading url 6/10]: http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc
    
    [Reading url 7/10]: http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/avg/ESPRESSO_Real-Time_v2_Averages_Best
    
    [Reading url 8/10]: http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/his/ESPRESSO_Real-Time_v2_History_Best
    
    [Reading url 9/10]: http://thredds.secoora.org/thredds/dodsC/G1_SST_GLOBAL.nc
    
    [Reading url 10/10]: http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global

</pre>
</div>
Next, we will match them with the nearest observed time-series. The `max_dist=0.08` is in degrees, that is roughly 8 kilometers.

<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
import iris
from iris.pandas import as_series
from ioos_tools.tardis import (make_tree, get_nearest_water,
                               add_station, ensure_timeseries, remove_ssh)

for mod_name, cube in cubes.items():
    fname = '{}.nc'.format(mod_name)
    fname = os.path.join(save_dir, fname)
    print(fmt(' Downloading to file {} '.format(fname)))
    try:
        tree, lon, lat = make_tree(cube)
    except CoordinateNotFoundError as e:
        print('Cannot make KDTree for: {}'.format(mod_name))
        continue
    # Get model series at observed locations.
    raw_series = dict()
    for obs in observations:
        obs = obs._metadata
        station = obs['station_code']
        try:
            kw = dict(k=10, max_dist=0.08, min_var=0.01)
            args = cube, tree, obs['lon'], obs['lat']
            try:
                series, dist, idx = get_nearest_water(*args, **kw)
            except RuntimeError as e:
                print('Cannot download {!r}.\n{}'.format(cube, e))
                series = None
        except ValueError as e:
            status = "No Data"
            print('[{}] {}'.format(status, obs['station_name']))
            continue
        if not series:
            status = "Land   "
        else:
            raw_series.update({station: series})
            series = as_series(series)
            status = "Water  "
        print('[{}] {}'.format(status, obs['station_name']))
    if raw_series:  # Save cube.
        for station, cube in raw_series.items():
            cube = add_station(cube, station)
            cube = remove_ssh(cube)
        try:
            cube = iris.cube.CubeList(raw_series.values()).merge_cube()
        except MergeError as e:
            print(e)
        ensure_timeseries(cube)
        try:
            iris.save(cube, fname)
        except AttributeError:
            # FIXME: we should patch the bad attribute instead of removing everything.
            cube.attributes = {}
            iris.save(cube, fname)
        del cube
    print('Finished processing [{}]'.format(mod_name))
```
<div class="output_area"><div class="prompt"></div>
<pre>
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    [Water  ] Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/FVCOM_Forecasts-NECOFS_GOM3_FORECAST.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    [Water  ] Boston, MA
    Finished processing [FVCOM_Forecasts-NECOFS_GOM3_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc 
    [No Data] BOSTON 16 NM East of Boston, MA
    [Land   ] Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_BOSTON_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc 
    [No Data] BOSTON 16 NM East of Boston, MA
    [No Data] Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/SECOORA_NCSU_CNAPS.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    [Land   ] Boston, MA
    Finished processing [SECOORA_NCSU_CNAPS]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/roms_2013_da_avg-ESPRESSO_Real-Time_v2_Averages_Best.nc 
    [Land   ] BOSTON 16 NM East of Boston, MA
    [Land   ] Boston, MA
    Finished processing [roms_2013_da_avg-ESPRESSO_Real-Time_v2_Averages_Best]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/roms_2013_da-ESPRESSO_Real-Time_v2_History_Best.nc 
    [Land   ] BOSTON 16 NM East of Boston, MA
    [Land   ] Boston, MA
    Finished processing [roms_2013_da-ESPRESSO_Real-Time_v2_History_Best]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/G1_SST_GLOBAL.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    [Water  ] Boston, MA
    Finished processing [G1_SST_GLOBAL]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/pacioos_hycom-global.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    [Land   ] Boston, MA
    Finished processing [pacioos_hycom-global]

</pre>
</div>
Now it is possible to compute some simple comparison metrics. First we'll calculate the model mean bias:

$$ \text{MB} = \mathbf{\overline{m}} - \mathbf{\overline{o}}$$

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
from ioos_tools.ioos import stations_keys


def rename_cols(df, config):
    cols = stations_keys(config, key='station_name')
    return df.rename(columns=cols)
```

<div class="prompt input_prompt">
In&nbsp;[16]:
</div>

```python
from ioos_tools.ioos import load_ncs
from ioos_tools.skill_score import mean_bias, apply_skill

dfs = load_ncs(config)

df = apply_skill(dfs, mean_bias, remove_mean=False, filter_tides=False)
skill_score = dict(mean_bias=df.to_dict())

# Filter out stations with no valid comparison.
df.dropna(how='all', axis=1, inplace=True)
df = df.applymap('{:.2f}'.format).replace('nan', '--')
```

And the root mean squared rrror of the deviations from the mean:
$$ \text{CRMS} = \sqrt{\left(\mathbf{m'} - \mathbf{o'}\right)^2}$$

where: $\mathbf{m'} = \mathbf{m} - \mathbf{\overline{m}}$ and $\mathbf{o'} = \mathbf{o} - \mathbf{\overline{o}}$

<div class="prompt input_prompt">
In&nbsp;[17]:
</div>

```python
from ioos_tools.skill_score import rmse

dfs = load_ncs(config)

df = apply_skill(dfs, rmse, remove_mean=True, filter_tides=False)
skill_score['rmse'] = df.to_dict()

# Filter out stations with no valid comparison.
df.dropna(how='all', axis=1, inplace=True)
df = df.applymap('{:.2f}'.format).replace('nan', '--')
```

The next 2 cells make the scores "pretty" for plotting.

<div class="prompt input_prompt">
In&nbsp;[18]:
</div>

```python
import pandas as pd

# Stringfy keys.
for key in skill_score.keys():
    skill_score[key] = {str(k): v for k, v in skill_score[key].items()}

mean_bias = pd.DataFrame.from_dict(skill_score['mean_bias'])
mean_bias = mean_bias.applymap('{:.2f}'.format).replace('nan', '--')

skill_score = pd.DataFrame.from_dict(skill_score['rmse'])
skill_score = skill_score.applymap('{:.2f}'.format).replace('nan', '--')
```

<div class="prompt input_prompt">
In&nbsp;[19]:
</div>

```python
from ioos_tools.ioos import make_map

bbox = config['region']['bbox']
units = config['units']
run_name = config['run_name']

kw = dict(zoom_start=11, line=True, states=False,
          secoora_stations=False, layers=False)
mapa = make_map(bbox, **kw)
```

The cells from `[20]` to `[25]` create a [`folium`](https://github.com/python-visualization/folium) map with [`bokeh`](http://bokeh.pydata.org/en/latest/) for the time-series at the observed points.

Note that we did mark the nearest model cell location used in the comparison.

<div class="prompt input_prompt">
In&nbsp;[20]:
</div>

```python
all_obs = stations_keys(config)

from glob import glob
from operator import itemgetter

import iris
import folium
from folium.plugins import MarkerCluster

iris.FUTURE.netcdf_promote = True

big_list = []
for fname in glob(os.path.join(save_dir, "*.nc")):
    if 'OBS_DATA' in fname:
        continue
    cube = iris.load_cube(fname)
    model = os.path.split(fname)[1].split('-')[-1].split('.')[0]
    lons = cube.coord(axis='X').points
    lats = cube.coord(axis='Y').points
    stations = cube.coord('station_code').points
    models = [model]*lons.size
    lista = zip(models, lons.tolist(), lats.tolist(), stations.tolist())
    big_list.extend(lista)

big_list.sort(key=itemgetter(3))
df = pd.DataFrame(big_list, columns=['name', 'lon', 'lat', 'station'])
df.set_index('station', drop=True, inplace=True)
groups = df.groupby(df.index)


locations, popups = [], []
for station, info in groups:
    sta_name = all_obs[station]
    for lat, lon, name in zip(info.lat, info.lon, info.name):
        locations.append([lat, lon])
        popups.append('[{}]: {}'.format(name, sta_name))

MarkerCluster(locations=locations, popups=popups).add_to(mapa)
```




    <folium.plugins.marker_cluster.MarkerCluster at 0x7fe76371beb8>



Here we use a dictionary with some models we expect to find so we can create a better legend for the plots. If any new models are found, we will use its filename in the legend as a default until we can go back and add a short name to our library.

<div class="prompt input_prompt">
In&nbsp;[21]:
</div>

```python
titles = {
    'coawst_4_use_best': 'COAWST_4',
    'global': 'HYCOM',
    'NECOFS_GOM3_FORECAST': 'NECOFS_GOM3',
    'NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST': 'NECOFS_MassBay',
    'OBS_DATA': 'Observations'
}
```

<div class="prompt input_prompt">
In&nbsp;[22]:
</div>

```python
from bokeh.resources import CDN
from bokeh.plotting import figure
from bokeh.embed import file_html
from bokeh.models import HoverTool
from itertools import cycle
from bokeh.palettes import Spectral6

from folium import IFrame

# Plot defaults.
colors = Spectral6
colorcycler = cycle(colors)
tools = "pan,box_zoom,reset"
width, height = 750, 250


def make_plot(df, station):
    p = figure(toolbar_location="above",
               x_axis_type="datetime",
               width=width,
               height=height,
               tools=tools,
               title=str(station))
    for column, series in df.iteritems():
        series.dropna(inplace=True)
        if not series.empty:
            line = p.line(
                x=series.index,
                y=series.values,
                legend="%s" % titles.get(column, column),
                line_color=next(colorcycler),
                line_width=5,
                line_cap='round',
                line_join='round'
            )
            if 'OBS_DATA' not in column:
                bias = mean_bias[str(station)][column]
                skill = skill_score[str(station)][column]
            else:
                skill = bias = 'NA'
            p.add_tools(HoverTool(tooltips=[("Name", "%s" % column),
                                            ("Bias", bias),
                                            ("Skill", skill)],
                                  renderers=[line]))
    return p


def make_marker(p, station):
    lons = stations_keys(config, key='lon')
    lats = stations_keys(config, key='lat')

    lon, lat = lons[station], lats[station]
    html = file_html(p, CDN, station)
    iframe = IFrame(html, width=width+40, height=height+80)

    popup = folium.Popup(iframe, max_width=2650)
    icon = folium.Icon(color='green', icon='stats')
    marker = folium.Marker(location=[lat, lon],
                           popup=popup,
                           icon=icon)
    return marker
```

<div class="prompt input_prompt">
In&nbsp;[23]:
</div>

```python
dfs = load_ncs(config)

for station in dfs:
    sta_name = all_obs[station]
    df = dfs[station]
    if df.empty:
        continue
    p = make_plot(df, station)
    maker = make_marker(p, station)
    maker.add_to(mapa)

mapa
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVMgPSBmYWxzZTsgTF9OT19UT1VDSCA9IGZhbHNlOyBMX0RJU0FCTEVfM0QgPSBmYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL3VucGtnLmNvbS9sZWFmbGV0QDEuMC4xL2Rpc3QvbGVhZmxldC5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9hamF4Lmdvb2dsZWFwaXMuY29tL2FqYXgvbGlicy9qcXVlcnkvMS4xMS4xL2pxdWVyeS5taW4uanMiPjwvc2NyaXB0PgogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vYm9vdHN0cmFwLzMuMi4wL2pzL2Jvb3RzdHJhcC5taW4uanMiPjwvc2NyaXB0PgogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL0xlYWZsZXQuYXdlc29tZS1tYXJrZXJzLzIuMC4yL2xlYWZsZXQuYXdlc29tZS1tYXJrZXJzLmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMS4wLjAvbGVhZmxldC5tYXJrZXJjbHVzdGVyLXNyYy5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvbGVhZmxldC5tYXJrZXJjbHVzdGVyLzEuMC4wL2xlYWZsZXQubWFya2VyY2x1c3Rlci5qcyI+PC9zY3JpcHQ+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vdW5wa2cuY29tL2xlYWZsZXRAMS4wLjEvZGlzdC9sZWFmbGV0LmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC5taW4uY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMS4wLjAvTWFya2VyQ2x1c3Rlci5EZWZhdWx0LmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvbGVhZmxldC5tYXJrZXJjbHVzdGVyLzEuMC4wL01hcmtlckNsdXN0ZXIuY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhd2dpdC5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIiAvPgogICAgPHN0eWxlPmh0bWwsIGJvZHkge3dpZHRoOiAxMDAlO2hlaWdodDogMTAwJTttYXJnaW46IDA7cGFkZGluZzogMDt9PC9zdHlsZT4KICAgIDxzdHlsZT4jbWFwIHtwb3NpdGlvbjphYnNvbHV0ZTt0b3A6MDtib3R0b206MDtyaWdodDowO2xlZnQ6MDt9PC9zdHlsZT4KICAgIAogICAgICAgICAgICA8c3R5bGU+ICNtYXBfNjkzOWUxN2MyODY5NDI5ZGIwMzIzNWFlMDE0Y2VlOGIgewogICAgICAgICAgICAgICAgcG9zaXRpb24gOiByZWxhdGl2ZTsKICAgICAgICAgICAgICAgIHdpZHRoIDogMTAwLjAlOwogICAgICAgICAgICAgICAgaGVpZ2h0OiAxMDAuMCU7CiAgICAgICAgICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgICAgICAgICAgdG9wOiAwLjAlOwogICAgICAgICAgICAgICAgfQogICAgICAgICAgICA8L3N0eWxlPgogICAgICAgIAogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8xLjAuMC9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMS4wLjAvTWFya2VyQ2x1c3Rlci5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8xLjAuMC9NYXJrZXJDbHVzdGVyLkRlZmF1bHQuY3NzIiAvPgo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgICAgICAgICA8ZGl2IGNsYXNzPSJmb2xpdW0tbWFwIiBpZD0ibWFwXzY5MzllMTdjMjg2OTQyOWRiMDMyMzVhZTAxNGNlZThiIiA+PC9kaXY+CiAgICAgICAgCjwvYm9keT4KPHNjcmlwdD4gICAgCiAgICAKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIHNvdXRoV2VzdCA9IEwubGF0TG5nKC05MCwgLTE4MCk7CiAgICAgICAgICAgICAgICB2YXIgbm9ydGhFYXN0ID0gTC5sYXRMbmcoOTAsIDE4MCk7CiAgICAgICAgICAgICAgICB2YXIgYm91bmRzID0gTC5sYXRMbmdCb3VuZHMoc291dGhXZXN0LCBub3J0aEVhc3QpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIHZhciBtYXBfNjkzOWUxN2MyODY5NDI5ZGIwMzIzNWFlMDE0Y2VlOGIgPSBMLm1hcCgKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICdtYXBfNjkzOWUxN2MyODY5NDI5ZGIwMzIzNWFlMDE0Y2VlOGInLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAge2NlbnRlcjogWzQyLjMzLC03MC45MzVdLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgem9vbTogMTEsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBtYXhCb3VuZHM6IGJvdW5kcywKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGxheWVyczogW10sCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB3b3JsZENvcHlKdW1wOiBmYWxzZSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGNyczogTC5DUlMuRVBTRzM4NTcKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgIAogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciB0aWxlX2xheWVyXzFhMTgxZjIxY2U3YzQwMjFhMzMwNDAyZDlmNGVhNzE0ID0gTC50aWxlTGF5ZXIoCiAgICAgICAgICAgICAgICAnaHR0cHM6Ly97c30udGlsZS5vcGVuc3RyZWV0bWFwLm9yZy97en0ve3h9L3t5fS5wbmcnLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIG1heFpvb206IDE4LAogICAgICAgICAgICAgICAgICAgIG1pblpvb206IDEsCiAgICAgICAgICAgICAgICAgICAgY29udGludW91c1dvcmxkOiBmYWxzZSwKICAgICAgICAgICAgICAgICAgICBub1dyYXA6IGZhbHNlLAogICAgICAgICAgICAgICAgICAgIGF0dHJpYnV0aW9uOiAnRGF0YSBieSA8YSBocmVmPSJodHRwOi8vb3BlbnN0cmVldG1hcC5vcmciPk9wZW5TdHJlZXRNYXA8L2E+LCB1bmRlciA8YSBocmVmPSJodHRwOi8vd3d3Lm9wZW5zdHJlZXRtYXAub3JnL2NvcHlyaWdodCI+T0RiTDwvYT4uJywKICAgICAgICAgICAgICAgICAgICBkZXRlY3RSZXRpbmE6IGZhbHNlCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfNjkzOWUxN2MyODY5NDI5ZGIwMzIzNWFlMDE0Y2VlOGIpOwoKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgbWFjcm9fZWxlbWVudF9iYmJmYThjYzIwOWY0NGRiYjg3ZGUxYzI1NWEzNTlhYSA9IEwudGlsZUxheWVyLndtcygKICAgICAgICAgICAgICAgICdodHRwOi8vaGZybmV0LnVjc2QuZWR1L3RocmVkZHMvd21zL0hGUk5ldC9VU0VHQy82a20vaG91cmx5L1JUVicsCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgbGF5ZXJzOiAnc3VyZmFjZV9zZWFfd2F0ZXJfdmVsb2NpdHknLAogICAgICAgICAgICAgICAgICAgIHN0eWxlczogJycsCiAgICAgICAgICAgICAgICAgICAgZm9ybWF0OiAnaW1hZ2UvcG5nJywKICAgICAgICAgICAgICAgICAgICB0cmFuc3BhcmVudDogdHJ1ZSwKICAgICAgICAgICAgICAgICAgICB2ZXJzaW9uOiAnMS4xLjEnLAogICAgICAgICAgICAgICAgICAgICBhdHRyaWJ1dGlvbjogJ0hGUk5ldCcKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApLmFkZFRvKG1hcF82OTM5ZTE3YzI4Njk0MjlkYjAzMjM1YWUwMTRjZWU4Yik7CgogICAgICAgIAogICAgCiAgICAgICAgICAgICAgICB2YXIgcG9seV9saW5lX2I1NGQxN2FiNmNiNjRlMjFhNWIwOTY1OGJjZWFlYzA4ID0gTC5wb2x5bGluZSgKICAgICAgICAgICAgICAgICAgICBbWzQyLjAzMDAwMDAwMDAwMDAwMSwgLTcxLjI5OTk5OTk5OTk5OTk5N10sIFs0Mi4wMzAwMDAwMDAwMDAwMDEsIC03MC41Njk5OTk5OTk5OTk5OTNdLCBbNDIuNjMwMDAwMDAwMDAwMDAzLCAtNzAuNTY5OTk5OTk5OTk5OTkzXSwgWzQyLjYzMDAwMDAwMDAwMDAwMywgLTcxLjI5OTk5OTk5OTk5OTk5N10sIFs0Mi4wMzAwMDAwMDAwMDAwMDEsIC03MS4yOTk5OTk5OTk5OTk5OTddXSwKICAgICAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgICAgIGNvbG9yOiAnI0ZGMDAwMCcsCiAgICAgICAgICAgICAgICAgICAgICAgIHdlaWdodDogMiwKICAgICAgICAgICAgICAgICAgICAgICAgb3BhY2l0eTogMC45LAogICAgICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcF82OTM5ZTE3YzI4Njk0MjlkYjAzMjM1YWUwMTRjZWU4Yi5hZGRMYXllcihwb2x5X2xpbmVfYjU0ZDE3YWI2Y2I2NGUyMWE1YjA5NjU4YmNlYWVjMDgpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgbGF5ZXJfY29udHJvbF8xNTUwOWVjMDQyZWU0NDBlYjkwNGY4MzkyMzc4ZWRkNSA9IHsKICAgICAgICAgICAgICAgIGJhc2VfbGF5ZXJzIDogeyAib3BlbnN0cmVldG1hcCIgOiB0aWxlX2xheWVyXzFhMTgxZjIxY2U3YzQwMjFhMzMwNDAyZDlmNGVhNzE0LCB9LAogICAgICAgICAgICAgICAgb3ZlcmxheXMgOiB7ICJIRiBSYWRhciIgOiBtYWNyb19lbGVtZW50X2JiYmZhOGNjMjA5ZjQ0ZGJiODdkZTFjMjU1YTM1OWFhLCB9CiAgICAgICAgICAgICAgICB9OwogICAgICAgICAgICBMLmNvbnRyb2wubGF5ZXJzKAogICAgICAgICAgICAgICAgbGF5ZXJfY29udHJvbF8xNTUwOWVjMDQyZWU0NDBlYjkwNGY4MzkyMzc4ZWRkNS5iYXNlX2xheWVycywKICAgICAgICAgICAgICAgIGxheWVyX2NvbnRyb2xfMTU1MDllYzA0MmVlNDQwZWI5MDRmODM5MjM3OGVkZDUub3ZlcmxheXMsCiAgICAgICAgICAgICAgICB7cG9zaXRpb246ICd0b3ByaWdodCcsCiAgICAgICAgICAgICAgICAgY29sbGFwc2VkOiB0cnVlLAogICAgICAgICAgICAgICAgIGF1dG9aSW5kZXg6IHRydWUKICAgICAgICAgICAgICAgIH0pLmFkZFRvKG1hcF82OTM5ZTE3YzI4Njk0MjlkYjAzMjM1YWUwMTRjZWU4Yik7CiAgICAgICAgCiAgICAKICAgICAgICAgICAgICAgIHZhciBtYXJrZXJfY2x1c3Rlcl85NWVjZTk1NjhkY2Q0YmRkYjI4ZDI4NzYyNDA4NjAxNSA9IEwubWFya2VyQ2x1c3Rlckdyb3VwKCk7CiAgICAgICAgICAgICAgICBtYXBfNjkzOWUxN2MyODY5NDI5ZGIwMzIzNWFlMDE0Y2VlOGIuYWRkTGF5ZXIobWFya2VyX2NsdXN0ZXJfOTVlY2U5NTY4ZGNkNGJkZGIyOGQyODc2MjQwODYwMTUpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl8xMDgyZjI3Y2I4MjQ0ZWVmYTEzNzAzYzZlOGRhNTAxNyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzQyLjM0MTMyNzY2NzIsLTcwLjY0ODMxNTQyOTddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcmtlcl9jbHVzdGVyXzk1ZWNlOTU2OGRjZDRiZGRiMjhkMjg3NjI0MDg2MDE1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzBkZjZmYWVmYzg1YzQxY2ZhYTNmODg4MjY4YTAwMDcyID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzM0NDk2YTA1ZTg4ZDQzMjRiYmFiYjMwOGY2OTg5ZWZiID0gJCgnPGRpdiBpZD0iaHRtbF8zNDQ5NmEwNWU4OGQ0MzI0YmJhYmIzMDhmNjk4OWVmYiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+W05FQ09GU19GVkNPTV9PQ0VBTl9NQVNTQkFZX0ZPUkVDQVNUXTogQk9TVE9OIDE2IE5NIEVhc3Qgb2YgQm9zdG9uLCBNQTwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfMGRmNmZhZWZjODVjNDFjZmFhM2Y4ODgyNjhhMDAwNzIuc2V0Q29udGVudChodG1sXzM0NDk2YTA1ZTg4ZDQzMjRiYmFiYjMwOGY2OTg5ZWZiKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfMTA4MmYyN2NiODI0NGVlZmExMzcwM2M2ZThkYTUwMTcuYmluZFBvcHVwKHBvcHVwXzBkZjZmYWVmYzg1YzQxY2ZhYTNmODg4MjY4YTAwMDcyKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2ViNWRkZTA1ZThjMDQzMWJiYzU4MGMwNzEwNmYzZDhiID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNDIuMzQzNzg0MzMyMywtNzAuNjUyNDU4MTkwOV0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFya2VyX2NsdXN0ZXJfOTVlY2U5NTY4ZGNkNGJkZGIyOGQyODc2MjQwODYwMTUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfMWQ3MmZhZjg2ZmY1NDlmMmIyYmY2YTZhMDEyNWMzZmYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYjUwZjg0ZGUwMmJhNDUzMWE5ZmYyMWJlMjYwMzg1YzQgPSAkKCc8ZGl2IGlkPSJodG1sX2I1MGY4NGRlMDJiYTQ1MzFhOWZmMjFiZTI2MDM4NWM0IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5bTkVDT0ZTX0dPTTNfRk9SRUNBU1RdOiBCT1NUT04gMTYgTk0gRWFzdCBvZiBCb3N0b24sIE1BPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8xZDcyZmFmODZmZjU0OWYyYjJiZjZhNmEwMTI1YzNmZi5zZXRDb250ZW50KGh0bWxfYjUwZjg0ZGUwMmJhNDUzMWE5ZmYyMWJlMjYwMzg1YzQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9lYjVkZGUwNWU4YzA0MzFiYmM1ODBjMDcxMDZmM2Q4Yi5iaW5kUG9wdXAocG9wdXBfMWQ3MmZhZjg2ZmY1NDlmMmIyYmY2YTZhMDEyNWMzZmYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfMzkzZGNiODQ3NDQ3NDU4MGE0YjRjYzUwZTM4MTc3MTEgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zNTM3MDI4MzA0LC03MC42NDE0MDI3MTQ4XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXJrZXJfY2x1c3Rlcl85NWVjZTk1NjhkY2Q0YmRkYjI4ZDI4NzYyNDA4NjAxNSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8yZDM1MTY4ZGFkNTI0NjM1YWRkNGVjNTE0ZWJiNWRlZCA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF84MzhlMmU1NGY4OWE0YWEwODAwOTA4OWNlOTdmYjBhMSA9ICQoJzxkaXYgaWQ9Imh0bWxfODM4ZTJlNTRmODlhNGFhMDgwMDkwODljZTk3ZmIwYTEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPltTRUNPT1JBX05DU1VfQ05BUFNdOiBCT1NUT04gMTYgTk0gRWFzdCBvZiBCb3N0b24sIE1BPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF8yZDM1MTY4ZGFkNTI0NjM1YWRkNGVjNTE0ZWJiNWRlZC5zZXRDb250ZW50KGh0bWxfODM4ZTJlNTRmODlhNGFhMDgwMDkwODljZTk3ZmIwYTEpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl8zOTNkY2I4NDc0NDc0NTgwYTRiNGNjNTBlMzgxNzcxMS5iaW5kUG9wdXAocG9wdXBfMmQzNTE2OGRhZDUyNDYzNWFkZDRlYzUxNGViYjVkZWQpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfY2RhMjQ2YzczYTZmNGM4OTk3ZDZjNjc5Y2Y2YWE3NjQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zNDUwMDEyMjA3LC03MC42NTQ5OTg3NzkzXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXJrZXJfY2x1c3Rlcl85NWVjZTk1NjhkY2Q0YmRkYjI4ZDI4NzYyNDA4NjAxNSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF8zZWMwZTIyMDdjM2E0ZTJkOTFhYTY0MDY3Yjg5OGM0NyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF85M2ZmZjk4ZWMxMGM0Mzg0YWU3NzQ2YmFmM2JlN2ExMiA9ICQoJzxkaXYgaWQ9Imh0bWxfOTNmZmY5OGVjMTBjNDM4NGFlNzc0NmJhZjNiZTdhMTIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPltHMV9TU1RfR0xPQkFMXTogQk9TVE9OIDE2IE5NIEVhc3Qgb2YgQm9zdG9uLCBNQTwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfM2VjMGUyMjA3YzNhNGUyZDkxYWE2NDA2N2I4OThjNDcuc2V0Q29udGVudChodG1sXzkzZmZmOThlYzEwYzQzODRhZTc3NDZiYWYzYmU3YTEyKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfY2RhMjQ2YzczYTZmNGM4OTk3ZDZjNjc5Y2Y2YWE3NjQuYmluZFBvcHVwKHBvcHVwXzNlYzBlMjIwN2MzYTRlMmQ5MWFhNjQwNjdiODk4YzQ3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2I4NGY1YzU1MTBkYzQyNmNhMmI2ODIzZTVlY2EwNzY5ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNDIuMzI0OCwtNzAuNjM5OTddLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcmtlcl9jbHVzdGVyXzk1ZWNlOTU2OGRjZDRiZGRiMjhkMjg3NjI0MDg2MDE1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2U1NGI2YmZmZGZlZTQwYTY4YmQyZDQ0NWNlOGJmMDRmID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzk2N2M0NjY5NTE0ZDRjNjM5Y2QxODM4ZTJmZWM4NmEwID0gJCgnPGRpdiBpZD0iaHRtbF85NjdjNDY2OTUxNGQ0YzYzOWNkMTgzOGUyZmVjODZhMCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+W2dsb2JhbF06IEJPU1RPTiAxNiBOTSBFYXN0IG9mIEJvc3RvbiwgTUE8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2U1NGI2YmZmZGZlZTQwYTY4YmQyZDQ0NWNlOGJmMDRmLnNldENvbnRlbnQoaHRtbF85NjdjNDY2OTUxNGQ0YzYzOWNkMTgzOGUyZmVjODZhMCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2I4NGY1YzU1MTBkYzQyNmNhMmI2ODIzZTVlY2EwNzY5LmJpbmRQb3B1cChwb3B1cF9lNTRiNmJmZmRmZWU0MGE2OGJkMmQ0NDVjZThiZjA0Zik7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9mNzdlNjczYTIyMmY0NDUzOTlkOTE1OGMxOWM1ZGIwZiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzQyLjM1MzQ1MDc3NTEsLTcxLjA1MTMxNTMwNzZdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcmtlcl9jbHVzdGVyXzk1ZWNlOTU2OGRjZDRiZGRiMjhkMjg3NjI0MDg2MDE1KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwX2NmMzZmNzljNjcwMDQ0ODI4NTQ4MzJhZDFkNTQzY2VjID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2MzYjU4OWVlMWIyYzQ0MDRiZWY5YjU2MjE4NTM0MDlhID0gJCgnPGRpdiBpZD0iaHRtbF9jM2I1ODllZTFiMmM0NDA0YmVmOWI1NjIxODUzNDA5YSIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+W05FQ09GU19GVkNPTV9PQ0VBTl9NQVNTQkFZX0ZPUkVDQVNUXTogQm9zdG9uLCBNQTwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfY2YzNmY3OWM2NzAwNDQ4Mjg1NDgzMmFkMWQ1NDNjZWMuc2V0Q29udGVudChodG1sX2MzYjU4OWVlMWIyYzQ0MDRiZWY5YjU2MjE4NTM0MDlhKTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZjc3ZTY3M2EyMjJmNDQ1Mzk5ZDkxNThjMTljNWRiMGYuYmluZFBvcHVwKHBvcHVwX2NmMzZmNzljNjcwMDQ0ODI4NTQ4MzJhZDFkNTQzY2VjKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2I5ODZhMWRlZDJkNzRiOWM4OTY1NTNjMjlkYzQzZmE5ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNDIuMzU2NTcxMTk3NSwtNzEuMDQ2ODQ0NDgyNF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFya2VyX2NsdXN0ZXJfOTVlY2U5NTY4ZGNkNGJkZGIyOGQyODc2MjQwODYwMTUpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZTBlNTRhZDViYmFmNDA0Y2IxZmViNTg4NzQ1NTg2M2IgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfMzAxM2IxZGMxYmQ5NDlkOWIyNzVhMzY4NTYzMTdiZDYgPSAkKCc8ZGl2IGlkPSJodG1sXzMwMTNiMWRjMWJkOTQ5ZDliMjc1YTM2ODU2MzE3YmQ2IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5bTkVDT0ZTX0dPTTNfRk9SRUNBU1RdOiBCb3N0b24sIE1BPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9lMGU1NGFkNWJiYWY0MDRjYjFmZWI1ODg3NDU1ODYzYi5zZXRDb250ZW50KGh0bWxfMzAxM2IxZGMxYmQ5NDlkOWIyNzVhMzY4NTYzMTdiZDYpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9iOTg2YTFkZWQyZDc0YjljODk2NTUzYzI5ZGM0M2ZhOS5iaW5kUG9wdXAocG9wdXBfZTBlNTRhZDViYmFmNDA0Y2IxZmViNTg4NzQ1NTg2M2IpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZjI0OTYzY2RhNDU0NDFiMmJjZmIwZjIxZjYyMzJiNTUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zNjUwMDE2Nzg1LC03MS4wNTUwMDAzMDUyXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXJrZXJfY2x1c3Rlcl85NWVjZTk1NjhkY2Q0YmRkYjI4ZDI4NzYyNDA4NjAxNSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF81YTQ3OGZlYWYxZmU0ZWRmOTg2YzA0NGUzZjQ4NGJiNyA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF84ZDhjMDI1YmRlMmE0Mzg0YTIzY2Y0ZmY3NzVjODg2OCA9ICQoJzxkaXYgaWQ9Imh0bWxfOGQ4YzAyNWJkZTJhNDM4NGEyM2NmNGZmNzc1Yzg4NjgiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPltHMV9TU1RfR0xPQkFMXTogQm9zdG9uLCBNQTwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfNWE0NzhmZWFmMWZlNGVkZjk4NmMwNDRlM2Y0ODRiYjcuc2V0Q29udGVudChodG1sXzhkOGMwMjViZGUyYTQzODRhMjNjZjRmZjc3NWM4ODY4KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfZjI0OTYzY2RhNDU0NDFiMmJjZmIwZjIxZjYyMzJiNTUuYmluZFBvcHVwKHBvcHVwXzVhNDc4ZmVhZjFmZTRlZGY5ODZjMDQ0ZTNmNDg0YmI3KTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyXzU3ODg0YmEzYWQ5OTRiYmJiOTYzZTQ3NzM4NDI2NDMxID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNDIuMzQ2LC03MC42NTFdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcF82OTM5ZTE3YzI4Njk0MjlkYjAzMjM1YWUwMTRjZWU4Yik7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMWY5YTg1OGMyMTZmNDFlZmJjZDJkYzY0YmFiOTY1OTQgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdzdGF0cycsCiAgICAgICAgICAgICAgICAgICAgaWNvbkNvbG9yOiAnd2hpdGUnLAogICAgICAgICAgICAgICAgICAgIG1hcmtlckNvbG9yOiAnZ3JlZW4nLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTc4ODRiYTNhZDk5NGJiYmI5NjNlNDc3Mzg0MjY0MzEuc2V0SWNvbihpY29uXzFmOWE4NThjMjE2ZjQxZWZiY2QyZGM2NGJhYjk2NTk0KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzU3YTgyNDJkZTJhMjRmMzA4ZjEwZGQ5MzgzOGQ4MTA1ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICcyNjUwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaV9mcmFtZV80ODhmMTViNjE5NDk0MjJjYThjYTMwNTg1NjU0YTgwMiA9ICQoJzxpZnJhbWUgc3JjPSJkYXRhOnRleHQvaHRtbDtjaGFyc2V0PXV0Zi04O2Jhc2U2NCxDaUFnSUNBS1BDRkVUME5VV1ZCRklHaDBiV3crQ2p4b2RHMXNJR3hoYm1jOUltVnVJajRLSUNBZ0lEeG9aV0ZrUGdvZ0lDQWdJQ0FnSUR4dFpYUmhJR05vWVhKelpYUTlJblYwWmkwNElqNEtJQ0FnSUNBZ0lDQThkR2wwYkdVK05EUXdNVE04TDNScGRHeGxQZ29nSUNBZ0lDQWdJQW84YkdsdWF5QnlaV3c5SW5OMGVXeGxjMmhsWlhRaUlHaHlaV1k5SW1oMGRIQnpPaTh2WTJSdUxuQjVaR0YwWVM1dmNtY3ZZbTlyWldndmNtVnNaV0Z6WlM5aWIydGxhQzB3TGpFeUxqVXViV2x1TG1OemN5SWdkSGx3WlQwaWRHVjRkQzlqYzNNaUlDOCtDaUFnSUNBZ0lDQWdDanh6WTNKcGNIUWdkSGx3WlQwaWRHVjRkQzlxWVhaaGMyTnlhWEIwSWlCemNtTTlJbWgwZEhCek9pOHZZMlJ1TG5CNVpHRjBZUzV2Y21jdlltOXJaV2d2Y21Wc1pXRnpaUzlpYjJ0bGFDMHdMakV5TGpVdWJXbHVMbXB6SWo0OEwzTmpjbWx3ZEQ0S1BITmpjbWx3ZENCMGVYQmxQU0owWlhoMEwycGhkbUZ6WTNKcGNIUWlQZ29nSUNBZ1FtOXJaV2d1YzJWMFgyeHZaMTlzWlhabGJDZ2lhVzVtYnlJcE93bzhMM05qY21sd2RENEtJQ0FnSUNBZ0lDQThjM1I1YkdVK0NpQWdJQ0FnSUNBZ0lDQm9kRzFzSUhzS0lDQWdJQ0FnSUNBZ0lDQWdkMmxrZEdnNklERXdNQ1U3Q2lBZ0lDQWdJQ0FnSUNBZ0lHaGxhV2RvZERvZ01UQXdKVHNLSUNBZ0lDQWdJQ0FnSUgwS0lDQWdJQ0FnSUNBZ0lHSnZaSGtnZXdvZ0lDQWdJQ0FnSUNBZ0lDQjNhV1IwYURvZ09UQWxPd29nSUNBZ0lDQWdJQ0FnSUNCb1pXbG5hSFE2SURFd01DVTdDaUFnSUNBZ0lDQWdJQ0FnSUcxaGNtZHBiam9nWVhWMGJ6c0tJQ0FnSUNBZ0lDQWdJSDBLSUNBZ0lDQWdJQ0E4TDNOMGVXeGxQZ29nSUNBZ1BDOW9aV0ZrUGdvZ0lDQWdQR0p2WkhrK0NpQWdJQ0FnSUNBZ0NpQWdJQ0FnSUNBZ1BHUnBkaUJqYkdGemN6MGlZbXN0Y205dmRDSStDaUFnSUNBZ0lDQWdJQ0FnSUR4a2FYWWdZMnhoYzNNOUltSnJMWEJzYjNSa2FYWWlJR2xrUFNKaU4yRmpOV1ppTVMweU1XWTVMVFEwT0RrdE9UaGxPQzB6WWpneE4yWm1OV1F5TkdZaVBqd3ZaR2wyUGdvZ0lDQWdJQ0FnSUR3dlpHbDJQZ29nSUNBZ0lDQWdJQW9nSUNBZ0lDQWdJRHh6WTNKcGNIUWdkSGx3WlQwaWRHVjRkQzlxWVhaaGMyTnlhWEIwSWo0S0lDQWdJQ0FnSUNBZ0lDQWdLR1oxYm1OMGFXOXVLQ2tnZXdvZ0lDQWdJQ0FnSUNBZ2RtRnlJR1p1SUQwZ1puVnVZM1JwYjI0b0tTQjdDaUFnSUNBZ0lDQWdJQ0FnSUVKdmEyVm9Mbk5oWm1Wc2VTaG1kVzVqZEdsdmJpZ3BJSHNLSUNBZ0lDQWdJQ0FnSUNBZ0lDQjJZWElnWkc5amMxOXFjMjl1SUQwZ2V5SmxOVEJoTldVMlpDMDVORGt5TFRSaU1HUXRPVFUwTXkxbVl6bGpNVEV6T0RJMU9UTWlPbnNpY205dmRITWlPbnNpY21WbVpYSmxibU5sY3lJNlczc2lZWFIwY21saWRYUmxjeUk2ZTMwc0ltbGtJam9pTURJeE16QTNZbVF0TWpjeE5DMDBaR1UxTFRrNU5XWXRNVEJrWWpFNFpEUmxZVEF3SWl3aWRIbHdaU0k2SWxSdmIyeEZkbVZ1ZEhNaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWJXRjRYMmx1ZEdWeWRtRnNJam8xTURBdU1Dd2liblZ0WDIxcGJtOXlYM1JwWTJ0eklqb3dmU3dpYVdRaU9pSmhZVFEyTmpWak1pMWhOelEzTFRSaVltWXRPR1UyWmkxaU1UVTVaamc1T0RoaFltSWlMQ0owZVhCbElqb2lRV1JoY0hScGRtVlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWJHbHVaVjloYkhCb1lTSTZleUoyWVd4MVpTSTZNQzR4ZlN3aWJHbHVaVjlqWVhBaU9pSnliM1Z1WkNJc0lteHBibVZmWTI5c2IzSWlPbnNpZG1Gc2RXVWlPaUlqTVdZM04ySTBJbjBzSW14cGJtVmZhbTlwYmlJNkluSnZkVzVrSWl3aWJHbHVaVjkzYVdSMGFDSTZleUoyWVd4MVpTSTZOWDBzSW5naU9uc2labWxsYkdRaU9pSjRJbjBzSW5raU9uc2labWxsYkdRaU9pSjVJbjE5TENKcFpDSTZJbU00WTJNeE5HSTNMVGxsWmpVdE5EZGtNQzFpWmpVNUxUaGhNamMyTVRBeVlqVm1NU0lzSW5SNWNHVWlPaUpNYVc1bEluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteHBibVZmWVd4d2FHRWlPbnNpZG1Gc2RXVWlPakF1TVgwc0lteHBibVZmWTJGd0lqb2ljbTkxYm1RaUxDSnNhVzVsWDJOdmJHOXlJanA3SW5aaGJIVmxJam9pSXpGbU56ZGlOQ0o5TENKc2FXNWxYMnB2YVc0aU9pSnliM1Z1WkNJc0lteHBibVZmZDJsa2RHZ2lPbnNpZG1Gc2RXVWlPalY5TENKNElqcDdJbVpwWld4a0lqb2llQ0o5TENKNUlqcDdJbVpwWld4a0lqb2llU0o5ZlN3aWFXUWlPaUppTnpVeE4yRm1PQzFpTnpVM0xUUXlNRGN0T1RsbU5TMDNabVZrWVdNeE5tSXlNVE1pTENKMGVYQmxJam9pVEdsdVpTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SndiRzkwSWpwN0ltbGtJam9pTVdGak4yWTJaVEV0TkdZM01pMDBOakppTFdKa00ySXRPVEl6WkdJNVpURTBNVE01SWl3aWMzVmlkSGx3WlNJNklrWnBaM1Z5WlNJc0luUjVjR1VpT2lKUWJHOTBJbjBzSW5ScFkydGxjaUk2ZXlKcFpDSTZJbUUxTURZd016TXpMVGc1TlRNdE5EQTNZaTA1WWpnNExUSTRNRGRoTVdNNE4yUXhOaUlzSW5SNWNHVWlPaUpFWVhSbGRHbHRaVlJwWTJ0bGNpSjlmU3dpYVdRaU9pSmpaR1kzT0dJMU1TMHdZamM1TFRRMVpEWXRZbVZpTlMwd01XSXlNRGsyTUdOa09EQWlMQ0owZVhCbElqb2lSM0pwWkNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZTMwc0ltbGtJam9pWVRGa05UazVZbVF0WXpNd09DMDBNbVEwTFRnek5ESXRabUUxTXpGak5qUmpOVGxoSWl3aWRIbHdaU0k2SWtKaGMybGpWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltTmhiR3hpWVdOcklqcHVkV3hzZlN3aWFXUWlPaUkzTUdZME5tVTRZUzAxWVdWa0xUUm1PV1l0T1dNME15MWtaRFF3WldFMU1ERmpNak1pTENKMGVYQmxJam9pUkdGMFlWSmhibWRsTVdRaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWJHbHVaVjlqWVhBaU9pSnliM1Z1WkNJc0lteHBibVZmWTI5c2IzSWlPbnNpZG1Gc2RXVWlPaUlqWm1NNFpEVTVJbjBzSW14cGJtVmZhbTlwYmlJNkluSnZkVzVrSWl3aWJHbHVaVjkzYVdSMGFDSTZleUoyWVd4MVpTSTZOWDBzSW5naU9uc2labWxsYkdRaU9pSjRJbjBzSW5raU9uc2labWxsYkdRaU9pSjVJbjE5TENKcFpDSTZJalk1TXpnNE5qRXpMV1psWVRNdE5ESmpNeTA0TWpJekxXSXlPVFZpWkRObVlXUXlNQ0lzSW5SNWNHVWlPaUpNYVc1bEluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltUmhkR0ZmYzI5MWNtTmxJanA3SW1sa0lqb2labVppTXpCaU1qWXRaVEU0TkMwME16a3dMV0kwWXpZdE9XUXdOVFF4WVdaaU9HSmlJaXdpZEhsd1pTSTZJa052YkhWdGJrUmhkR0ZUYjNWeVkyVWlmU3dpWjJ4NWNHZ2lPbnNpYVdRaU9pSTVNMkl4TlRJd01TMDFOVGd4TFRReE5qa3RZbUl6TWkxaFltWTJPR1E0TnpOak9ETWlMQ0owZVhCbElqb2lUR2x1WlNKOUxDSm9iM1psY2w5bmJIbHdhQ0k2Ym5Wc2JDd2liWFYwWldSZloyeDVjR2dpT201MWJHd3NJbTV2Ym5ObGJHVmpkR2x2Ymw5bmJIbHdhQ0k2ZXlKcFpDSTZJbU00WTJNeE5HSTNMVGxsWmpVdE5EZGtNQzFpWmpVNUxUaGhNamMyTVRBeVlqVm1NU0lzSW5SNWNHVWlPaUpNYVc1bEluMHNJbk5sYkdWamRHbHZibDluYkhsd2FDSTZiblZzYkgwc0ltbGtJam9pTUdFeU5URXlPR0V0TnpaallpMDBORGc0TFdJNU5qRXRaRFl5T0RVNFlqUmtZMlJtSWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liR0ZpWld3aU9uc2lkbUZzZFdVaU9pSk9SVU5QUmxOZlIwOU5NeUo5TENKeVpXNWtaWEpsY25NaU9sdDdJbWxrSWpvaU1HRXlOVEV5T0dFdE56WmpZaTAwTkRnNExXSTVOakV0WkRZeU9EVTRZalJrWTJSbUlpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlYxOUxDSnBaQ0k2SWprMU9EYzJNR1JtTFdFMU1UUXRORGsxTmkxaE9EYzFMV0UwT0RCbE1HSTRabVZpWWlJc0luUjVjR1VpT2lKTVpXZGxibVJKZEdWdEluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltTmhiR3hpWVdOcklqcHVkV3hzTENKamIyeDFiVzVmYm1GdFpYTWlPbHNpZUNJc0lua2lYU3dpWkdGMFlTSTZleUo0SWpwN0lsOWZibVJoY25KaGVWOWZJam9pUVVGQ1FXcFBOakprVlVsQlFVTnFOemhpV2pGUlowRkJSVWR5TVhSdVZrTkJRVVEwTWxCcE1tUlZTVUZCVDBKSUwweGFNVkZuUVVGNVRHSXZkRzVXUTBGQlEzZEtVVTh6WkZWSlFVRkthVlZDY21ReFVXZEJRV2RCVFV0ME0xWkRRVUZDYjJObk1qTmtWVWxCUVVaRWFFVk1aREZSWjBGQlQwWkJWWFF6VmtOQlFVRm5kbmhsTTJSVlNVRkJRV2QxUnpka01WRm5RVUU0U25kbGRETldRMEZCUkZsRGVVc3paRlZKUVVGTlFqWktZbVF4VVdkQlFYRlBhMjkwTTFaRFFVRkRVVmREZVROa1ZVbEJRVWhxU0V3M1pERlJaMEZCV1VSWmVuUXpWa05CUVVKSmNGUmhNMlJWU1VGQlJFRlZUM0prTVZGblFVRkhTVTA1ZEROV1EwRkJRVUU0YTBNelpGVkpRVUZQYUdkU1RHUXhVV2RCUVRCTk9VaDBNMVpEUVVGRE5GQnJkVE5rVlVsQlFVdERkRlJ5WkRGUlowRkJhVUo0VTNRelZrTkJRVUozYVRGWE0yUlZTVUZCUm1vMlYweGtNVkZuUVVGUlIyeGpkRE5XUTBGQlFXOHlSaXN6WkZWSlFVRkNRa2haTjJReFVXZEJRU3RNVm0xME0xWkRRVUZFWjBwSGNUTmtWVWxCUVUxcFZHSmlaREZSWjBGQmMwRktlSFF6VmtOQlFVTlpZMWhUTTJSVlNVRkJTVVJuWkRka01WRm5RVUZoUlRrM2RETldRMEZCUWxGMmJqWXpaRlZKUVVGRVozUm5jbVF4VVdkQlFVbEtlVVowTTFaRFFVRkJTVU0wYlROa1ZVbEJRVkJDTldwTVpERlJaMEZCTWs5cFVIUXpWa05CUVVSQlZqVlBNMlJWU1VGQlMycEhiSEprTVZGblFVRnJSRmRoZEROV1EwRkJRalJ3U2pJelpGVkpRVUZIUVZSdlltUXhVV2RCUVZOSlMydDBNMVpEUVVGQmR6aGhaVE5rVlVsQlFVSm9aM0UzWkRGUlowRkJRVTByZFhRelZrTkJRVVJ2VUdKTE0yUlZTVUZCVGtOemRHSmtNVkZuUVVGMVFuVTFkRE5XUTBGQlEyZHBjbmt6WkZWSlFVRkphalYyTjJReFVXZEJRV05IYWtSME0xWkRRVUZDV1RFNFlUTmtWVWxCUVVWQ1IzbHlaREZSWjBGQlMweFlUblF6VmtOQlFVRlJTazVITTJSVlNVRkJVR2xUTVV4a01WRm5RVUUwUVVoWmRETldRMEZCUkVsalRuVXpaRlZKUVVGTVJHWXpjbVF4VVdkQlFXMUZOMmwwTTFaRFFVRkRRWFpsVnpOa1ZVbEJRVWRuY3paaVpERlJaMEZCVlVwMmMzUXpWa05CUVVFMFEzWkRNMlJWU1VGQlEwSTFPRGRrTVZGblFVRkRUMm95ZEROV1EwRkJSSGRXZG5FelpGVkpRVUZPYWtZdlltUXhVV2RCUVhkRVVVSjFTRlpEUVVGRGIyOTNVelJrVlVsQlFVcEJVME5NYURGUlowRkJaVWxGVEhWSVZrTkJRVUpuT0VFMk5HUlZTVUZCUldobVJYSm9NVkZuUVVGTlRUUldkVWhXUTBGQlFWbFFVbTAwWkZWSlFVRkJRM05JVEdneFVXZEJRVFpDYjJkMVNGWkRRVUZFVVdsVFR6UmtWVWxCUVV4cU5FcHlhREZSWjBGQmIwZGpjWFZJVmtOQlFVTkpNV2t5TkdSVlNVRkJTRUpHVFdKb01WRm5RVUZYVEZFd2RVaFdRMEZCUWtGSmVtazBaRlZKUVVGRGFWTlBOMmd4VVdkQlFVVkJSUzkxU0ZaRFFVRkVOR0l3U3pSa1ZVbEJRVTlFWlZKaWFERlJaMEZCZVVVeFNuVklWa05CUVVOM2RrVjVOR1JWU1VGQlNtZHlWVXhvTVZGblFVRm5TbkJVZFVoV1EwRkJRbTlEVm1VMFpGVkpRVUZHUWpSWGNtZ3hVV2RCUVU5UFpHUjFTRlpEUVVGQloxWnRSelJrVlVsQlFVRnFSbHBNYURGUlowRkJPRVJPYjNWSVZrTkJRVVJaYjIxMU5HUlZTVUZCVFVGU1lqZG9NVkZuUVVGeFNVSjVkVWhXUTBGQlExRTNNMWMwWkZWSlFVRklhR1ZsWW1neFVXZEJRVmxOTVRoMVNGWkRRVUZDU1ZCSlF6UmtWVWxCUVVSRGNtYzNhREZSWjBGQlIwSnhTSFZJVmtOQlFVRkJhVmx4TkdSVlNVRkJUMm96YW1Kb01WRm5RVUV3UjJGU2RVaFdRMEZCUXpReFdsTTBaRlZKUVVGTFFrVnRUR2d4VVdkQlFXbE1UMkoxU0ZaRFFVRkNkMGx3S3pSa1ZVbEJRVVpwVW05eWFERlJaMEZCVVVGRGJYVklWa05CUVVGdllqWnROR1JWU1VGQlFrUmxja3hvTVZGblFVRXJSWGwzZFVoV1EwRkJSR2QxTjA4MFpGVkpRVUZOWjNGME4yZ3hVV2RCUVhOS2JUWjFTRlpEUVVGRFdVTk1OalJrVlVsQlFVbENNM2RpYURGUlowRkJZVTlpUlhWSVZrTkJRVUpSVm1OcE5HUlZTVUZCUkdwRmVUZG9NVkZuUVVGSlJGQlFkVWhXUTBGQlFVbHZkRXMwWkZWSlFVRlFRVkV4Y21neFVXZEJRVEpJTDFwMVNGWkRRVUZFUVRkMGVUUmtWVWxCUVV0b1pEUk1hREZSWjBGQmEwMTZhblZJVmtOQlFVSTBUeXRsTkdSVlNVRkJSME54Tm5Kb01WRm5RVUZUUW01MWRVaFdRMEZCUVhkcFVFYzBaRlZKUVVGQ2FqTTVUR2d4VVdkQlFVRkhZalIxU0ZaRFFVRkViekZRZFRSa1ZVbEJRVTVDUkM4M2FERlJaMEZCZFV4SlEzVllWa05CUVVOblNWRmhOV1JWU1VGQlNXbFJRMkpzTVZGblFVRmpVRGhOZFZoV1EwRkJRbGxpYUVNMVpGVkpRVUZGUkdSRk4yd3hVV2RCUVV0RmQxaDFXRlpEUVVGQlVYVjRjVFZrVlVsQlFWQm5jRWh5YkRGUlowRkJORXBuYUhWWVZrTkJRVVJKUW5sWE5XUlZTVUZCVEVJeVMweHNNVkZuUVVGdFQxVnlkVmhXUXlJc0ltUjBlWEJsSWpvaVpteHZZWFEyTkNJc0luTm9ZWEJsSWpwYk1UWTRYWDBzSW5raU9uc2lYMTl1WkdGeWNtRjVYMThpT2lKQlFVRkJRVWxqUzBjd1FVRkJRVVJCTURoallWRkJRVUZCUjBGbmFGSndRVUZCUVVGSlJ6RkRSMnRCUVVGQlFrRmlhR05oVVVGQlFVRkhRblkzUW14QlFVRkJRV2RJUkVKSFZVRkJRVUZDUVhwS2ExcFJRVUZCUVVGQmIyTm9iRUZCUVVGQmQwbE9TMGRWUVVGQlFVTkJZM3BWV2xGQlFVRkJSMEpxU1VKc1FVRkJRVUZKUmsxTVIxVkJRVUZCUVdkd2NYZGFVVUZCUVVGQlJEVlVVbkJCUVVGQlFVRkZlblpIYTBGQlFVRkRRVmRoZDJKUlFVRkJRVU5DYm1GU2VFRkJRVUZCYjBoUmJVaFZRVUZCUVVKbldWYzRZMUZCUVVGQlEwSlBkVUowUVVGQlFVRTBSRzlDUnpCQlFVRkJRMmR2UkUxaVVVRkJRVUZGUVVkYWFIUkJRVUZCUVVGSGVWbEhNRUZCUVVGQ1FTOHhUV0pSUVVGQlFVbERVMFI0ZEVGQlFVRkJkME5ZVEVkclFVRkJRVUpCVmxodllWRkJRVUZCUzBORlMxSndRVUZCUVVGSlRGUlpSMVZCUVVGQlEwRllXSGRhVVVGQlFVRk5RVWRKUW14QlFVRkJRVWxNUkVSSFJVRkJRVUZDUVRoTGExbFJRVUZCUVVkQmQydENhRUZCUVVGQlowaENNa2RGUVVGQlFVRkJLMUJaV1ZGQlFVRkJTVUl2Wkhoc1FVRkJRVUZCUVdZMFIxVkJRVUZCUTJkd2NXdGhVVUZCUVVGSFFrZFhlSFJCUVVGQlFVRlBXVTFJUlVGQlFVRkNRVEpvVldOUlFVRkJRVWxFVDBob2VFRkJRVUZCZDAxSmJraEZRVUZCUVVGblp6QkJZMUZCUVVGQlNVSkVWMUo0UVVGQlFVRTBRVTU1U0VWQlFVRkJRV2QxYkZWalVVRkJRVUZGUW5kUFVuaEJRVUZCUVdkRFdXUklSVUZCUVVGRFowWkJkMk5SUVVGQlFVMUJReXQ0ZEVGQlFVRkJORkJFY0Vjd1FVRkJRVU5uVjA5QllsRkJRVUZCU1VSQk1XaDBRVUZCUVVGUlEycE9SekJCUVVGQlFXYzNaRUZpVVVGQlFVRkRRM2t4UW5SQlFVRkJRVUZJWmxsSE1FRkJRVUZEWjJwSE5HTlJRVUZCUVVkRGFVSkNNVUZCUVVGQlFVeHBZVWhWUVVGQlFVUm5SRUpOWlZGQlFVRkJUMEpvYVhnMVFVRkJRVUYzVEZsRVNEQkJRVUZCUWtGRFFYZG1VVUZCUVVGTFFscEdRamxCUVVGQlFVbExjMk5JTUVGQlFVRkJRWGxVYjJWUlFVRkJRVUZFYmxkQ01VRkJRVUZCTkVGU00waEZRVUZCUVVKQlJqRlpZMUZCUVVGQlRVRndUbEo0UVVGQlFVRkpSSGRWU0VWQlFVRkJRV2N4ZG5OaVVVRkJRVUZCUW5jMGVIUkJRVUZCUVVGQmNreEhNRUZCUVVGQlFUWktOR0pSUVVGQlFVRkVSMk5vZEVGQlFVRkJRVXRTUjBjd1FVRkJRVVJuZWk5SllWRkJRVUZCVDBRM2JtaHdRVUZCUVVGM1EyUk1SMnRCUVVGQlJHZE5RekJoVVVGQlFVRlBRVFZFZUhCQlFVRkJRVUZGVUhoSFZVRkJRVUZFWnpOM1kyRlJRVUZCUVUxQ09FaG9jRUZCUVVGQmIwSnJNVWRyUVVGQlFVTkJkR2xOWVZGQlFVRkJTVUpVUldod1FVRkJRVUZaVUVGQlIydEJRVUZCUVdjM1QxbGhVVUZCUVVGTlJHNTZRblJCUVVGQlFXZFBUM2xJUlVGQlFVRkVaMFF6UVdOUlFVRkJRVWRCT0V4U2VFRkJRVUZCZDBkcWNVY3dRVUZCUVVOblUyTlZZbEZCUVVGQlIwRnhiMEowUVVGQlFVRlJRWFEzUnpCQlFVRkJRa0ZoTWxsaVVVRkJRVUZGUkV4VlVuUkJRVUZCUVZGRGN6bEhNRUZCUVVGQ1owSjVPR0pSUVVGQlFVdEVha2xDZEVGQlFVRkJkMHc0VTBjd1FVRkJRVU5CVDBNMFlsRkJRVUZCUlVONFUxSjBRVUZCUVVGQlEzQnNSekJCUVVGQlJFRTRNWGRpVVVGQlFVRkpRemxXUW5SQlFVRkJRVkZKWkUxSE1FRkJRVUZCWjBvNVNXRlJRVUZCUVVORVNGWjRjRUZCUVVGQlFVZG1aRWRWUVVGQlFVTm5iSGx2V2xGQlFVRkJSMFJKWkhob1FVRkJRVUZCVUc1RlJqQkJRVUZCUkdkNlJ6aFlVVUZCUVVGUFEyZEhhR1JCUVVGQlFYZElWRVpHYTBGQlFVRkVaMGsyYjFkUlFVRkJRVUZFVkdwb1drRkJRVUZCU1VsS2VrWnJRVUZCUVVGbmMyNUZWMUZCUVVGQlFVUnBZbmhhUVVGQlFVRkJRa3AxUm10QlFVRkJRa0UxYm5kWFVVRkJRVUZMUXpacGVGcEJRVUZCUVRSSk5tRkdhMEZCUVVGQ1oyZFNkMWhSUVVGQlFVRkNNRzVvWkVGQlFVRkJaMGRaWjBkRlFVRkJRVU5uU0dWM1dsRkJRVUZCVFVSVmRIaDBRVUZCUVVFMFNYVkVTRlZCUVVGQlJFRTJXRlZqVVVGQlFVRkpRa2hoUW5SQlFVRkJRVmxMVm1GSGEwRkJRVUZEUVc1d1dWcFJRVUZCUVVsRFdEQm9hRUZCUVVGQmIwcEJUMGRGUVVGQlFVTkJkR1JyV0ZGQlFVRkJTVVJoY0VKa1FVRkJRVUZaVURsMlJqQkJRVUZCUVVGck1qaFlVVUZCUVVGTFFXMWllR1JCUVVGQlFWRk1jSFZHTUVGQlFVRkVaemR1TUZoUlFVRkJRVWRCYW1wU1pFRkJRVUZCUVVacFkwWXdRVUZCUVVGQmVscGpXRkZCUVVGQlEwSkRhM2hrUVVGQlFVRkpUR1ZQUmpCQlFVRkJRVUY0U0dOWVVVRkJRVUZOUkZGWlFtUkJRVUZCUVc5T01VcEdNRUZCUVVGRFFUSnJaMWhSUVVGQlFVVkVXRko0WkVGQlFVRkJTVTVTUjBZd1FVRkJRVUpuYjJ4WldGRkJRVUZCU1VKM1dtaGtRVUZCUVVGM1JEVXlSakJCUVVGQlJFRlFibGxZVVVGQlFVRk5RU3RrYUdSQklpd2laSFI1Y0dVaU9pSm1iRzloZERZMElpd2ljMmhoY0dVaU9sc3hOamhkZlgxOUxDSnBaQ0k2SWpKbU1UWTNNakE0TFRGbU1EZ3RORE0yWVMxaFpEbG1MVGhoT0RGa1lqWXpOemc0TWlJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1KaGMyVWlPakkwTENKdFlXNTBhWE56WVhNaU9sc3hMRElzTkN3MkxEZ3NNVEpkTENKdFlYaGZhVzUwWlhKMllXd2lPalF6TWpBd01EQXdMakFzSW0xcGJsOXBiblJsY25aaGJDSTZNell3TURBd01DNHdMQ0p1ZFcxZmJXbHViM0pmZEdsamEzTWlPakI5TENKcFpDSTZJbVU0WlRSaFpqa3pMVEJqTjJJdE5EQmlZaTA0TjJNNExUZzVZV001TmpZMU5XWXlOeUlzSW5SNWNHVWlPaUpCWkdGd2RHbDJaVlJwWTJ0bGNpSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmtZWFJoWDNOdmRYSmpaU0k2ZXlKcFpDSTZJakptTVRZM01qQTRMVEZtTURndE5ETTJZUzFoWkRsbUxUaGhPREZrWWpZek56ZzRNaUlzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjBzSW1kc2VYQm9JanA3SW1sa0lqb2lOamt6T0RnMk1UTXRabVZoTXkwME1tTXpMVGd5TWpNdFlqSTVOV0prTTJaaFpESXdJaXdpZEhsd1pTSTZJa3hwYm1VaWZTd2lhRzkyWlhKZloyeDVjR2dpT201MWJHd3NJbTExZEdWa1gyZHNlWEJvSWpwdWRXeHNMQ0p1YjI1elpXeGxZM1JwYjI1ZloyeDVjR2dpT25zaWFXUWlPaUppTnpVeE4yRm1PQzFpTnpVM0xUUXlNRGN0T1RsbU5TMDNabVZrWVdNeE5tSXlNVE1pTENKMGVYQmxJam9pVEdsdVpTSjlMQ0p6Wld4bFkzUnBiMjVmWjJ4NWNHZ2lPbTUxYkd4OUxDSnBaQ0k2SWpVNE9EZ3pOVFprTFRNM016a3ROREl6WXkwNVpHSXpMV1kzT1RnMU5qYzBOV0k1TVNJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbUpoYzJVaU9qWXdMQ0p0WVc1MGFYTnpZWE1pT2xzeExESXNOU3d4TUN3eE5Td3lNQ3d6TUYwc0ltMWhlRjlwYm5SbGNuWmhiQ0k2TVRnd01EQXdNQzR3TENKdGFXNWZhVzUwWlhKMllXd2lPakV3TURBdU1Dd2liblZ0WDIxcGJtOXlYM1JwWTJ0eklqb3dmU3dpYVdRaU9pSmpNREZoTlRnM09TMWhZVE0xTFRRNFl6a3RZV013TlMwd1l6ZzBPREE0TkRNeVpUUWlMQ0owZVhCbElqb2lRV1JoY0hScGRtVlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVpHRjVjeUk2V3pFc05DdzNMREV3TERFekxERTJMREU1TERJeUxESTFMREk0WFgwc0ltbGtJam9pTm1ReE1HSmhZekl0TWprd01TMDBOVGhpTFdGa04yUXRNemxrTkRGa09EY3pZamxtSWl3aWRIbHdaU0k2SWtSaGVYTlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVpHRjVjeUk2V3pFc01pd3pMRFFzTlN3MkxEY3NPQ3c1TERFd0xERXhMREV5TERFekxERTBMREUxTERFMkxERTNMREU0TERFNUxESXdMREl4TERJeUxESXpMREkwTERJMUxESTJMREkzTERJNExESTVMRE13TERNeFhYMHNJbWxrSWpvaU9UYzFPR05rTW1VdFlXSTRNUzAwWWpBMUxUazJaV1l0T1RBelltVXpPVFJtTWpkbUlpd2lkSGx3WlNJNklrUmhlWE5VYVdOclpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liblZ0WDIxcGJtOXlYM1JwWTJ0eklqbzFmU3dpYVdRaU9pSmhOVEEyTURNek15MDRPVFV6TFRRd04ySXRPV0k0T0MweU9EQTNZVEZqT0Rka01UWWlMQ0owZVhCbElqb2lSR0YwWlhScGJXVlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVptOXliV0YwZEdWeUlqcDdJbWxrSWpvaVl6VTNaVEU1WldNdE56STJaUzAwTWpBekxUbG1NVEV0T1dJMU5EUmhNalZpTkRKaklpd2lkSGx3WlNJNklrSmhjMmxqVkdsamEwWnZjbTFoZEhSbGNpSjlMQ0p3Ykc5MElqcDdJbWxrSWpvaU1XRmpOMlkyWlRFdE5HWTNNaTAwTmpKaUxXSmtNMkl0T1RJelpHSTVaVEUwTVRNNUlpd2ljM1ZpZEhsd1pTSTZJa1pwWjNWeVpTSXNJblI1Y0dVaU9pSlFiRzkwSW4wc0luUnBZMnRsY2lJNmV5SnBaQ0k2SW1FeFpEVTVPV0prTFdNek1EZ3ROREprTkMwNE16UXlMV1poTlRNeFl6WTBZelU1WVNJc0luUjVjR1VpT2lKQ1lYTnBZMVJwWTJ0bGNpSjlmU3dpYVdRaU9pSXlOV015T0RrM09DMW1OREZoTFRSbE56Z3RZbVV4TXkwMU1HTXhZemd3Tm1GbE4yUWlMQ0owZVhCbElqb2lUR2x1WldGeVFYaHBjeUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpqWVd4c1ltRmpheUk2Ym5Wc2JDd2lZMjlzZFcxdVgyNWhiV1Z6SWpwYkluZ2lMQ0o1SWwwc0ltUmhkR0VpT25zaWVDSTZleUpmWDI1a1lYSnlZWGxmWHlJNklrRkJRa0ZxVHpZeVpGVkpRVUZEYWpjNFlsb3hVV2RCUVVWSGNqRjBibFpEUVVGQlFUaHJRek5rVlVsQlFVOW9aMUpNWkRGUlowRkJNRTA1U0hRelZrTkJRVVJCVmpWUE0yUlZTVUZCUzJwSGJISmtNVkZuUVVGclJGZGhkRE5XUTBGQlEwRjJaVmN6WkZWSlFVRkhaM00yWW1ReFVXZEJRVlZLZG5OME0xWkRJaXdpWkhSNWNHVWlPaUptYkc5aGREWTBJaXdpYzJoaGNHVWlPbHN4TWwxOUxDSjVJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lRVUZCUVZsRlltaEhNRUZCUVVGQ1ozZFVWV05SUVVGQlFVZEJPR2xvZUVGQlFVRkJVVWRtYlVsVlFVRkJRVU5uY0U1QmFGRkJRVUZCVDBSb2RXbEdRVUZCUVVGWlJUWTBTREJCUVVGQlFVRk5OMEZtVVVGQlFVRk5RVmh4UWpsQlFVRkJRVmxNTnpGSWEwRkJRVUZDWjNaMlZXVlJRVUZCUVVkREt6bFNOVUVpTENKa2RIbHdaU0k2SW1ac2IyRjBOalFpTENKemFHRndaU0k2V3pFeVhYMTlmU3dpYVdRaU9pSmtZbUU1T0dFd05DMDVPRFprTFRSa01tWXRPRGcyWmkxbU9UZzJNRGM1WWpCbVlqY2lMQ0owZVhCbElqb2lRMjlzZFcxdVJHRjBZVk52ZFhKalpTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmlaV3h2ZHlJNlczc2lhV1FpT2lKbE9XTmhORGN4TlMxaE56Z3hMVFJoTnpZdE9HRTNOUzFoWVdJeVpURmlOVGMyWW1NaUxDSjBlWEJsSWpvaVJHRjBaWFJwYldWQmVHbHpJbjFkTENKc1pXWjBJanBiZXlKcFpDSTZJakkxWXpJNE9UYzRMV1kwTVdFdE5HVTNPQzFpWlRFekxUVXdZekZqT0RBMllXVTNaQ0lzSW5SNWNHVWlPaUpNYVc1bFlYSkJlR2x6SW4xZExDSndiRzkwWDJobGFXZG9kQ0k2TWpVd0xDSndiRzkwWDNkcFpIUm9Jam8zTlRBc0luSmxibVJsY21WeWN5STZXM3NpYVdRaU9pSmxPV05oTkRjeE5TMWhOemd4TFRSaE56WXRPR0UzTlMxaFlXSXlaVEZpTlRjMlltTWlMQ0owZVhCbElqb2lSR0YwWlhScGJXVkJlR2x6SW4wc2V5SnBaQ0k2SW1Oa1pqYzRZalV4TFRCaU56a3RORFZrTmkxaVpXSTFMVEF4WWpJd09UWXdZMlE0TUNJc0luUjVjR1VpT2lKSGNtbGtJbjBzZXlKcFpDSTZJakkxWXpJNE9UYzRMV1kwTVdFdE5HVTNPQzFpWlRFekxUVXdZekZqT0RBMllXVTNaQ0lzSW5SNWNHVWlPaUpNYVc1bFlYSkJlR2x6SW4wc2V5SnBaQ0k2SWpnME56SXdNemM1TFdWbVpHVXROR05qTXkwNU5tVmtMVGhoTVRBd1lXRTNaalJoTmlJc0luUjVjR1VpT2lKSGNtbGtJbjBzZXlKcFpDSTZJalExTlRRMlpqRTNMV1poTmpBdE5EWTVZeTA0TmpJeExXTXlZakkzTlRJek1XTXpNeUlzSW5SNWNHVWlPaUpDYjNoQmJtNXZkR0YwYVc5dUluMHNleUpwWkNJNkltUXhNbU15WkRZekxXUXlNelV0TkRBMk9TMDVOV1ZrTFRFNFl6VXpZMlZqTTJVNU5DSXNJblI1Y0dVaU9pSk1aV2RsYm1RaWZTeDdJbWxrSWpvaVpXUmhNMlUxTVdZdE16Um1OUzAwWVRoaUxUazFNV1F0WXpCaU5tVXdNekl3T0RJNUlpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlN4N0ltbGtJam9pTkRBeU4yUTNOVFl0T0RjM01DMDBZVE0wTFdGa1pEVXRZakExTTJGbU16QmlNVFpsSWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmU3g3SW1sa0lqb2lNR0V5TlRFeU9HRXROelpqWWkwME5EZzRMV0k1TmpFdFpEWXlPRFU0WWpSa1kyUm1JaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZTeDdJbWxrSWpvaU1UWTJPVFJsTjJZdE1ETmpNQzAwWmpjMkxXRmlaR0l0WXpVeE1qazNZelJrWkRrd0lpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlN4N0ltbGtJam9pTlRnNE9ETTFObVF0TXpjek9TMDBNak5qTFRsa1lqTXRaamM1T0RVMk56UTFZamt4SWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmU3g3SW1sa0lqb2lZak5oTlRReE5XRXRZakkzWmkwMFlUZGxMV0l6T0dFdE1tUTNaREUwWW1NeFpqWTRJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZWMHNJblJwZEd4bElqcDdJbWxrSWpvaU5UZGlORFEwWWpVdE56QTVOQzAwTVRjMkxXSTBNamt0T1daaE1XTmpOR1V4WTJJM0lpd2lkSGx3WlNJNklsUnBkR3hsSW4wc0luUnZiMnhmWlhabGJuUnpJanA3SW1sa0lqb2lNREl4TXpBM1ltUXRNamN4TkMwMFpHVTFMVGs1TldZdE1UQmtZakU0WkRSbFlUQXdJaXdpZEhsd1pTSTZJbFJ2YjJ4RmRtVnVkSE1pZlN3aWRHOXZiR0poY2lJNmV5SnBaQ0k2SWpsak1HTmxabU5oTFdabFpXRXROR1JqWWkwNE1UUTFMV1l5T1dFM00ySXlOMk0yWlNJc0luUjVjR1VpT2lKVWIyOXNZbUZ5SW4wc0luUnZiMnhpWVhKZmJHOWpZWFJwYjI0aU9pSmhZbTkyWlNJc0luaGZjbUZ1WjJVaU9uc2lhV1FpT2lJM01HWTBObVU0WVMwMVlXVmtMVFJtT1dZdE9XTTBNeTFrWkRRd1pXRTFNREZqTWpNaUxDSjBlWEJsSWpvaVJHRjBZVkpoYm1kbE1XUWlmU3dpZVY5eVlXNW5aU0k2ZXlKcFpDSTZJbU15WlRsbE9HTXdMV1k0WVdFdE5HUXlOQzFoTjJRNExUWTNZamt3WVRNM1pHWTVPU0lzSW5SNWNHVWlPaUpFWVhSaFVtRnVaMlV4WkNKOWZTd2lhV1FpT2lJeFlXTTNaalpsTVMwMFpqY3lMVFEyTW1JdFltUXpZaTA1TWpOa1lqbGxNVFF4TXpraUxDSnpkV0owZVhCbElqb2lSbWxuZFhKbElpd2lkSGx3WlNJNklsQnNiM1FpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWTJGc2JHSmhZMnNpT201MWJHd3NJbU52YkhWdGJsOXVZVzFsY3lJNld5SjRJaXdpZVNKZExDSmtZWFJoSWpwN0luZ2lPbnNpWDE5dVpHRnljbUY1WDE4aU9pSkJRVVJCVmpWUE0yUlZTVUZCUzJwSGJISmtNVkZuUVVGclJGZGhkRE5XUTBGQlFqUndTakl6WkZWSlFVRkhRVlJ2WW1ReFVXZEJRVk5KUzJ0ME0xWkRRVUZCZHpoaFpUTmtWVWxCUVVKb1ozRTNaREZSWjBGQlFVMHJkWFF6VmtOQlFVUnZVR0pMTTJSVlNVRkJUa056ZEdKa01WRm5RVUYxUW5VMWRETldRMEZCUTJkcGNua3paRlZKUVVGSmFqVjJOMlF4VVdkQlFXTkhha1IwTTFaRFFVRkNXVEU0WVROa1ZVbEJRVVZDUjNseVpERlJaMEZCUzB4WVRuUXpWa05CUVVGUlNrNUhNMlJWU1VGQlVHbFRNVXhrTVZGblFVRTBRVWhaZEROV1EwRkJSRWxqVG5VelpGVkpRVUZNUkdZemNtUXhVV2RCUVcxRk4ybDBNMVpEUVVGRFFYWmxWek5rVlVsQlFVZG5jelppWkRGUlowRkJWVXAyYzNRelZrTkJRVUUwUTNaRE0yUlZTVUZCUTBJMU9EZGtNVkZuUVVGRFQyb3lkRE5XUTBGQlJIZFdkbkV6WkZWSlFVRk9ha1l2WW1ReFVXZEJRWGRFVVVKMVNGWkRRVUZEYjI5M1V6UmtWVWxCUVVwQlUwTk1hREZSWjBGQlpVbEZUSFZJVmtOQlFVSm5PRUUyTkdSVlNVRkJSV2htUlhKb01WRm5RVUZOVFRSV2RVaFdRMEZCUVZsUVVtMDBaRlZKUVVGQlEzTklUR2d4VVdkQlFUWkNiMmQxU0ZaRFFVRkVVV2xUVHpSa1ZVbEJRVXhxTkVweWFERlJaMEZCYjBkamNYVklWa05CUVVOSk1Xa3lOR1JWU1VGQlNFSkdUV0pvTVZGblFVRlhURkV3ZFVoV1EwRkJRa0ZKZW1rMFpGVkpRVUZEYVZOUE4yZ3hVV2RCUVVWQlJTOTFTRlpEUVVGRU5HSXdTelJrVlVsQlFVOUVaVkppYURGUlowRkJlVVV4U25WSVZrTkJRVU4zZGtWNU5HUlZTVUZCU21keVZVeG9NVkZuUVVGblNuQlVkVWhXUTBGQlFtOURWbVUwWkZWSlFVRkdRalJYY21neFVXZEJRVTlQWkdSMVNGWkRRVUZCWjFadFJ6UmtWVWxCUVVGcVJscE1hREZSWjBGQk9FUk9iM1ZJVmtOQlFVUlpiMjExTkdSVlNVRkJUVUZTWWpkb01WRm5RVUZ4U1VKNWRVaFdRMEZCUTFFM00xYzBaRlZKUVVGSWFHVmxZbWd4VVdkQlFWbE5NVGgxU0ZaRFFVRkNTVkJKUXpSa1ZVbEJRVVJEY21jM2FERlJaMEZCUjBKeFNIVklWa05CUVVGQmFWbHhOR1JWU1VGQlQyb3phbUpvTVZGblFVRXdSMkZTZFVoV1EwRkJRelF4V2xNMFpGVkpRVUZMUWtWdFRHZ3hVV2RCUVdsTVQySjFTRlpEUVVGQ2QwbHdLelJrVlVsQlFVWnBVbTl5YURGUlowRkJVVUZEYlhWSVZrTkJRVUZ2WWpadE5HUlZTVUZCUWtSbGNreG9NVkZuUVVFclJYbDNkVWhXUTBGQlJHZDFOMDgwWkZWSlFVRk5aM0YwTjJneFVXZEJRWE5LYlRaMVNGWkRRVUZEV1VOTU5qUmtWVWxCUVVsQ00zZGlhREZSWjBGQllVOWlSWFZJVmtOQlFVSlJWbU5wTkdSVlNVRkJSR3BGZVRkb01WRm5RVUZKUkZCUWRVaFdRMEZCUVVsdmRFczBaRlZKUVVGUVFWRXhjbWd4VVdkQlFUSklMMXAxU0ZaRFFVRkVRVGQwZVRSa1ZVbEJRVXRvWkRSTWFERlJaMEZCYTAxNmFuVklWa05CUVVJMFR5dGxOR1JWU1VGQlIwTnhObkpvTVZGblFVRlRRbTUxZFVoV1EwRkJRWGRwVUVjMFpGVkpRVUZDYWpNNVRHZ3hVV2RCUVVGSFlqUjFTRlpEUVVGRWJ6RlFkVFJrVlVsQlFVNUNSQzgzYURGUlowRkJkVXhKUTNWWVZrTkJRVU5uU1ZGaE5XUlZTVUZCU1dsUlEySnNNVkZuUVVGalVEaE5kVmhXUTBGQlFsbGlhRU0xWkZWSlFVRkZSR1JGTjJ3eFVXZEJRVXRGZDFoMVdGWkRRVUZCVVhWNGNUVmtWVWxCUVZCbmNFaHliREZSWjBGQk5FcG5hSFZZVmtOQlFVUkpRbmxYTldSVlNVRkJURUl5UzB4c01WRm5RVUZ0VDFWeWRWaFdRMEZCUTBGV1F5czFaRlZKUVVGSGFrUk5jbXd4VVdkQlFWVkVTVEoxV0ZaRFFVRkJORzlVYlRWa1ZVbEJRVU5CVVZCaWJERlJaMEZCUTBnNVFYVllWa05CUVVSM04xVlBOV1JWU1VGQlRtaGpVamRzTVZGblFVRjNUWFJMZFZoV1EwRkJRMjlQYXpZMVpGVkpRVUZLUTNCVlltd3hVV2RCUVdWQ2FGWjFXRlpEUVVGQ1oyZ3hhVFZrVlVsQlFVVnFNbGMzYkRGUlowRkJUVWRXWm5WWVZrTkJRVUZaTVVkTE5XUlZTVUZCUVVKRVduSnNNVkZuUVVFMlRFWndkVmhXUTBGQlJGRkpSekkxWkZWSlFVRk1hVkJqVEd3eFVXZEJRVzlRTlhwMVdGWkRRVUZEU1dKWVpUVmtWVWxCUVVoRVkyVnliREZSWjBGQlYwVjBLM1ZZVmtNaUxDSmtkSGx3WlNJNkltWnNiMkYwTmpRaUxDSnphR0Z3WlNJNld6RTBORjE5TENKNUlqcDdJbDlmYm1SaGNuSmhlVjlmSWpvaVFVRkJRWGRCTWtkSlJVRkJRVUZEWnpsdmEyZFJRVUZCUVVkRVptcFRRa0ZCUVVGQlVVMXBVa2xGUVVGQlFVSm5Xak52WjFGQlFVRkJSMEZIV1hsQ1FVRkJRVUZuUzFaTVNVVkJRVUZCUW1kSVJVRm5VVUZCUVVGSFExUk9RMEpCUVVGQlFWRkJiM0JKUlVGQlFVRkNRVTlTYzJkUlFVRkJRVWRDYjBSVFFrRkJRVUZCZDBNM0wwZ3dRVUZCUVVKQlJVTlJaMUZCUVVGQlFVTktVME5DUVVGQlFVRTBRVVowU1VWQlFVRkJRa0ZoU2pSblVVRkJRVUZKUkU5NmVVSkJRVUZCUVRSRVVVSkpWVUZCUVVGQlFXZDRUV2hSUVVGQlFVRkVVa3BUUmtGQlFVRkJTVUk0TkVsVlFVRkJRVUZuUTNsamFGRkJRVUZCUlVRelJsTkdRVUZCUVVGUlQwMUZTVlZCUVVGQlJHZEZWVEJvVVVGQlFVRkpRa0ZzVTBaQlFVRkJRVWxITDJSSlZVRkJRVUZEUVdKelNXaFJRVUZCUVUxQ2RIQjVSa0ZCUVVGQlNVY3lUVWxWUVVGQlFVTm5VMGM0YUZGQlFVRkJRVUZyVldsR1FVRkJRVUZuVURnd1NWVkJRVUZCUkdka1VuZG9VVUZCUVVGSFJITkJlVVpCUVVGQlFYZEhUSEpKUlVGQlFVRkJaMWQyU1dkUlFVRkJRVWxDVWl0VFFrRkJRVUZCTkVWblFVbFZRVUZCUVVGblF5OUJaMUZCUVVGQlIwUk9NM2xDUVVGQlFVRnZTUzlRU1VWQlFVRkJRVUV2YzNOblVVRkJRVUZIUW5ONVEwSkJRVUZCUVhkT2NrVkpSVUZCUVVGRFFVVmlSV2RSUVVGQlFVVkNTVzVUUWtGQlFVRkJRVWdyU2tsRlFVRkJRVUpuZDAxaloxRkJRVUZCUzBGQ1FtbEdRVUZCUVVGQlJVNUZTVlZCUVVGQlEwRkpVelJvVVVGQlFVRlBSQzlHZVVaQlFVRkJRVmxPTkVKSlZVRkJRVUZFUVRCUVdXZFJRVUZCUVVGRVJEWjVRa0ZCUVVGQldVeFlaMGxGUVVGQlFVSkJTamx6WjFGQlFVRkJRVU5hTVZOQ1FVRkJRVUUwUVhKUlNVVkJRVUZCUkdkSlpHOW5VVUZCUVVGUFFUUTFRMEpCUVVGQlFUUkZMM1ZKUlVGQlFVRkVaMjkyVVdkUlFVRkJRVUZFTWl0cFFrRkJRVUZCUVVWclFrbFZRVUZCUVVGQk1pdHJaMUZCUVVGQlFVSjBNR2xDUVVGQlFVRkJVQ3MyU1VWQlFVRkJSR2MxTlRSblVVRkJRVUZMUkZGbmFVSkJRVUZCUVdkTWJHMUpSVUZCUVVGRFFWVlViMmRSUVVGQlFVZEVjRVJUUWtGQlFVRkJiMEZNUkVnd1FVRkJRVU5uVEhOQlpsRkJRVUZCUzBKaGRsSTVRVUZCUVVGdlNXRTJTREJCUVVGQlFXZHFjbU5tVVVGQlFVRk5RMVowUWpsQlFVRkJRVkZLTW5oSU1FRkJRVUZCWjNKaFVXWlJRVUZCUVVGRE9XeDRPVUZCUVVGQk5FMTVTMGd3UVVGQlFVSm5MemhOWmxGQlFVRkJRVUY1TDFJNVFVRkJRVUZSUkVsaVNVVkJRVUZCUTJkUE0yZG5VVUZCUVVGRFFrWXhVMEpCUVVGQlFXZEZOSGxKVlVGQlFVRkJaek42TkdoUlFVRkJRVTlDZGxONVJrRkJRVUZCWjBGQ1dVbFZRVUZCUVVKblozcE5hRkZCUVVGQlIwRkhSSGxHUVVGQlFVRlJTVzV4U1VWQlFVRkJRMEZJZEdOblVVRkJRVUZOUTNwM2VVSkJRVUZCUVVGRmJYZEpSVUZCUVVGRVFYRkxRV2RSUVVGQlFVZEJTV3RUUWtGQlFVRkJTVWRwUWtsRlFVRkJRVVJuU1c1aloxRkJRVUZCU1VSa1lrTkNRVUZCUVVGUlNtaHBTVVZCUVVGQlFVRk9WVFJuVVVGQlFVRkxSRkpQVTBKQlFVRkJRVmxITkd4SlJVRkJRVUZEWjJkb1VXZFJRVUZCUVU5RFYwRjVRa0ZCUVVGQldVWmliRWd3UVVGQlFVSkJVRGMwWmxGQlFVRkJRMEZ2YkhnNVFVRkJRVUZCUWtaM1NEQkJRVUZCUWtGRVZHdG1VVUZCUVVGTFFVcEJhRGxCUVVGQlFUUkJXRXhJYTBGQlFVRkNRVkkxUldWUlFVRkJRVTFEU1ZaNE5VRkJRVUZCU1UxdlpFaHJRVUZCUVVOQk4wNDRaRkZCUVVGQlFVRlFiMmd4UVVGQlFVRlpSRVpyU0ZWQlFVRkJRMEZ3Vld0a1VVRkJRVUZKUVZwTWVERkJRVUZCUVc5Sk1GVklWVUZCUVVGRFp6RnJRV1JSUVVGQlFVbEJabUpTTVVGQlFVRkJaMGRwV2toVlFVRkJRVVJCYkU1alpGRkJRVUZCUVVSQ1JsSTFRVUZCUVVGUlR6RlVTR3RCUVVGQlFtZEZWMjlsVVVGQlFVRkxRVEZuUWpWQlFVRkJRWGRHYlZkSWEwRkJRVUZFWjFOeFRXVlJRVUZCUVU5Qk4zTkNOVUZCUVVGQlFVTXlPVWhyUVVGQlFVRkJZbkl3WlZGQlFVRkJUME4xZGxJMVFVRkJRVUUwVHlzNVNHdEJRVUZCUkdjM056QmxVVUZCUVVGUFJIWjJValZCSWl3aVpIUjVjR1VpT2lKbWJHOWhkRFkwSWl3aWMyaGhjR1VpT2xzeE5EUmRmWDE5TENKcFpDSTZJbVptWWpNd1lqSTJMV1V4T0RRdE5ETTVNQzFpTkdNMkxUbGtNRFUwTVdGbVlqaGlZaUlzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbVJwYldWdWMybHZiaUk2TVN3aWNHeHZkQ0k2ZXlKcFpDSTZJakZoWXpkbU5tVXhMVFJtTnpJdE5EWXlZaTFpWkROaUxUa3lNMlJpT1dVeE5ERXpPU0lzSW5OMVluUjVjR1VpT2lKR2FXZDFjbVVpTENKMGVYQmxJam9pVUd4dmRDSjlMQ0owYVdOclpYSWlPbnNpYVdRaU9pSmhNV1ExT1RsaVpDMWpNekE0TFRReVpEUXRPRE0wTWkxbVlUVXpNV00yTkdNMU9XRWlMQ0owZVhCbElqb2lRbUZ6YVdOVWFXTnJaWElpZlgwc0ltbGtJam9pT0RRM01qQXpOemt0Wldaa1pTMDBZMk16TFRrMlpXUXRPR0V4TURCaFlUZG1OR0UySWl3aWRIbHdaU0k2SWtkeWFXUWlmU3g3SW1GMGRISnBZblYwWlhNaU9udDlMQ0pwWkNJNklqUmhZelF6TldNNExUWmpZVE10TkRoak5pMWlNREU1TFRBM09UQmtNRGcyT0dNd1lpSXNJblI1Y0dVaU9pSkVZWFJsZEdsdFpWUnBZMnRHYjNKdFlYUjBaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdsdVpWOWpZWEFpT2lKeWIzVnVaQ0lzSW14cGJtVmZZMjlzYjNJaU9uc2lkbUZzZFdVaU9pSWpaVFptTlRrNEluMHNJbXhwYm1WZmFtOXBiaUk2SW5KdmRXNWtJaXdpYkdsdVpWOTNhV1IwYUNJNmV5SjJZV3gxWlNJNk5YMHNJbmdpT25zaVptbGxiR1FpT2lKNEluMHNJbmtpT25zaVptbGxiR1FpT2lKNUluMTlMQ0pwWkNJNklqa3pZakUxTWpBeExUVTFPREV0TkRFMk9TMWlZak15TFdGaVpqWTRaRGczTTJNNE15SXNJblI1Y0dVaU9pSk1hVzVsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3ZlN3aWFXUWlPaUk1WVdaa1pEVTFPQzA0WldJd0xUUmhaR1l0WW1VellpMDVOR0ZsWkdNMU9UTTNNamdpTENKMGVYQmxJam9pV1dWaGNuTlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW5Cc2IzUWlPbnNpYVdRaU9pSXhZV00zWmpabE1TMDBaamN5TFRRMk1tSXRZbVF6WWkwNU1qTmtZamxsTVRReE16a2lMQ0p6ZFdKMGVYQmxJam9pUm1sbmRYSmxJaXdpZEhsd1pTSTZJbEJzYjNRaWZTd2ljbVZ1WkdWeVpYSnpJanBiZXlKcFpDSTZJakJoTWpVeE1qaGhMVGMyWTJJdE5EUTRPQzFpT1RZeExXUTJNamcxT0dJMFpHTmtaaUlzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMWRMQ0owYjI5c2RHbHdjeUk2VzFzaVRtRnRaU0lzSWs1RlEwOUdVMTlIVDAwelgwWlBVa1ZEUVZOVUlsMHNXeUpDYVdGeklpd2lNQzQyTmlKZExGc2lVMnRwYkd3aUxDSXdMakkzSWwxZGZTd2lhV1FpT2lKbU1tSTNZVE14WXkwNE9HSTRMVFJtT0RZdE9HRmxOeTB4WWpnME4yUmtPVGxsTW1FaUxDSjBlWEJsSWpvaVNHOTJaWEpVYjI5c0luMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltRmpkR2wyWlY5a2NtRm5Jam9pWVhWMGJ5SXNJbUZqZEdsMlpWOXpZM0p2Ykd3aU9pSmhkWFJ2SWl3aVlXTjBhWFpsWDNSaGNDSTZJbUYxZEc4aUxDSjBiMjlzY3lJNlczc2lhV1FpT2lJME56UXhNVGRqWmkweE1HTXdMVFF5WlRrdE9UQTVNUzB5WVdWbVlUa3paRE5tWXpRaUxDSjBlWEJsSWpvaVVHRnVWRzl2YkNKOUxIc2lhV1FpT2lJeVlUYzFZMlpsTnkxaFl6ZzRMVFJrWlRZdFlqZ3laaTB5TkRabU9UUTRaVGM0WWpRaUxDSjBlWEJsSWpvaVFtOTRXbTl2YlZSdmIyd2lmU3g3SW1sa0lqb2lOekkyTmpZME0ySXROVE0wWkMwME9HWmhMV0U0TmpRdE1tSXdNbVJrWkdVME5tTmpJaXdpZEhsd1pTSTZJbEpsYzJWMFZHOXZiQ0o5TEhzaWFXUWlPaUkzTmpjNVl6ZzBPQzFqTURjMUxUUmpZMkl0T0RCbE15MDJOMk00WldWak9UUXlZamNpTENKMGVYQmxJam9pU0c5MlpYSlViMjlzSW4wc2V5SnBaQ0k2SWpNNVlUVXlZak0xTFdKbU1qUXROREJtTVMxaE1XWXlMVGxtTXpjMllXRTVOekl3TVNJc0luUjVjR1VpT2lKSWIzWmxjbFJ2YjJ3aWZTeDdJbWxrSWpvaVpqSmlOMkV6TVdNdE9EaGlPQzAwWmpnMkxUaGhaVGN0TVdJNE5EZGtaRGs1WlRKaElpd2lkSGx3WlNJNklraHZkbVZ5Vkc5dmJDSjlMSHNpYVdRaU9pSTNaVFU0TTJVeVl5MWxNelZoTFRRME1ETXRZakF3T1MxaVl6QXpZbVk1TmpZeE1ETWlMQ0owZVhCbElqb2lTRzkyWlhKVWIyOXNJbjBzZXlKcFpDSTZJbUprTkRVNVpXTTJMVFZqWXpZdE5EVTVNUzA0WkRJNUxXWXlNbU5tTlRjeE1EWTJZeUlzSW5SNWNHVWlPaUpJYjNabGNsUnZiMndpZlN4N0ltbGtJam9pTmpBNE5HUmpORFV0WkdZek15MDBaV1ZrTFRnd1pEZ3RObU0xT1RVM016WXpObUpoSWl3aWRIbHdaU0k2SWtodmRtVnlWRzl2YkNKOVhYMHNJbWxrSWpvaU9XTXdZMlZtWTJFdFptVmxZUzAwWkdOaUxUZ3hORFV0WmpJNVlUY3pZakkzWXpabElpd2lkSGx3WlNJNklsUnZiMnhpWVhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT250OUxDSnBaQ0k2SW1NMU4yVXhPV1ZqTFRjeU5tVXROREl3TXkwNVpqRXhMVGxpTlRRMFlUSTFZalF5WXlJc0luUjVjR1VpT2lKQ1lYTnBZMVJwWTJ0R2IzSnRZWFIwWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVpHRjBZVjl6YjNWeVkyVWlPbnNpYVdRaU9pSmtabUV6TnpCbE9DMWxaV0kwTFRReE5EQXRZbUl6TXkwNE1XVmhZakEwTkdFeU9XSWlMQ0owZVhCbElqb2lRMjlzZFcxdVJHRjBZVk52ZFhKalpTSjlMQ0puYkhsd2FDSTZleUpwWkNJNkltVmpNREUzWkRSaExUZzBOell0TkdZMFppMWhZakk1TFRVd01tWmpaV05tWkdVNU9TSXNJblI1Y0dVaU9pSk1hVzVsSW4wc0ltaHZkbVZ5WDJkc2VYQm9JanB1ZFd4c0xDSnRkWFJsWkY5bmJIbHdhQ0k2Ym5Wc2JDd2libTl1YzJWc1pXTjBhVzl1WDJkc2VYQm9JanA3SW1sa0lqb2lNVE0wWVdRNVkyTXRNelpsWVMwMFpXSXpMVGs0TXpNdFl6aGxNRFl3WVRCaFlUQmpJaXdpZEhsd1pTSTZJa3hwYm1VaWZTd2ljMlZzWldOMGFXOXVYMmRzZVhCb0lqcHVkV3hzZlN3aWFXUWlPaUppTTJFMU5ERTFZUzFpTWpkbUxUUmhOMlV0WWpNNFlTMHlaRGRrTVRSaVl6Rm1OamdpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmpZV3hzWW1GamF5STZiblZzYkN3aWNHeHZkQ0k2ZXlKcFpDSTZJakZoWXpkbU5tVXhMVFJtTnpJdE5EWXlZaTFpWkROaUxUa3lNMlJpT1dVeE5ERXpPU0lzSW5OMVluUjVjR1VpT2lKR2FXZDFjbVVpTENKMGVYQmxJam9pVUd4dmRDSjlMQ0p5Wlc1a1pYSmxjbk1pT2x0N0ltbGtJam9pWWpOaE5UUXhOV0V0WWpJM1ppMDBZVGRsTFdJek9HRXRNbVEzWkRFMFltTXhaalk0SWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmVjBzSW5SdmIyeDBhWEJ6SWpwYld5Sk9ZVzFsSWl3aVoyeHZZbUZzSWwwc1d5SkNhV0Z6SWl3aUxUQXVNellpWFN4YklsTnJhV3hzSWl3aU1DNHlPU0pkWFgwc0ltbGtJam9pTmpBNE5HUmpORFV0WkdZek15MDBaV1ZrTFRnd1pEZ3RObU0xT1RVM016WXpObUpoSWl3aWRIbHdaU0k2SWtodmRtVnlWRzl2YkNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKallXeHNZbUZqYXlJNmJuVnNiQ3dpWTI5c2RXMXVYMjVoYldWeklqcGJJbmdpTENKNUlsMHNJbVJoZEdFaU9uc2llQ0k2ZXlKZlgyNWtZWEp5WVhsZlh5STZJa0ZCUkVGV05VOHpaRlZKUVVGTGFrZHNjbVF4VVdkQlFXdEVWMkYwTTFaRFFVRkNOSEJLTWpOa1ZVbEJRVWRCVkc5aVpERlJaMEZCVTBsTGEzUXpWa05CUVVGM09HRmxNMlJWU1VGQlFtaG5jVGRrTVZGblFVRkJUU3QxZEROV1EwRkJSRzlRWWtzelpGVkpRVUZPUTNOMFltUXhVV2RCUVhWQ2RUVjBNMVpEUVVGRFoybHllVE5rVlVsQlFVbHFOWFkzWkRGUlowRkJZMGRxUkhRelZrTkJRVUpaTVRoaE0yUlZTVUZCUlVKSGVYSmtNVkZuUVVGTFRGaE9kRE5XUTBGQlFWRktUa2N6WkZWSlFVRlFhVk14VEdReFVXZEJRVFJCU0ZsME0xWkRRVUZFU1dOT2RUTmtWVWxCUVV4RVpqTnlaREZSWjBGQmJVVTNhWFF6VmtOQlFVTkJkbVZYTTJSVlNVRkJSMmR6Tm1Ka01WRm5RVUZWU25aemRETldRMEZCUVRSRGRrTXpaRlZKUVVGRFFqVTROMlF4VVdkQlFVTlBhakowTTFaRFFVRkVkMVoyY1ROa1ZVbEJRVTVxUmk5aVpERlJaMEZCZDBSUlFuVklWa05CUVVOdmIzZFROR1JWU1VGQlNrRlRRMHhvTVZGblFVRmxTVVZNZFVoV1EwRkJRbWM0UVRZMFpGVkpRVUZGYUdaRmNtZ3hVV2RCUVUxTk5GWjFTRlpEUVVGQldWQlNiVFJrVlVsQlFVRkRjMGhNYURGUlowRkJOa0p2WjNWSVZrTkJRVVJSYVZOUE5HUlZTVUZCVEdvMFNuSm9NVkZuUVVGdlIyTnhkVWhXUTBGQlEwa3hhVEkwWkZWSlFVRklRa1pOWW1neFVXZEJRVmRNVVRCMVNGWkRRVUZDUVVsNmFUUmtWVWxCUVVOcFUwODNhREZSWjBGQlJVRkZMM1ZJVmtOQlFVUTBZakJMTkdSVlNVRkJUMFJsVW1Kb01WRm5RVUY1UlRGS2RVaFdRMEZCUTNkMlJYazBaRlZKUVVGS1ozSlZUR2d4VVdkQlFXZEtjRlIxU0ZaRFFVRkNiME5XWlRSa1ZVbEJRVVpDTkZkeWFERlJaMEZCVDA5a1pIVklWa05CUVVGblZtMUhOR1JWU1VGQlFXcEdXa3hvTVZGblFVRTRSRTV2ZFVoV1EwRkJSRmx2YlhVMFpGVkpRVUZOUVZKaU4yZ3hVV2RCUVhGSlFubDFTRlpEUVVGRFVUY3pWelJrVlVsQlFVaG9aV1ZpYURGUlowRkJXVTB4T0hWSVZrTkJRVUpKVUVsRE5HUlZTVUZCUkVOeVp6ZG9NVkZuUVVGSFFuRklkVWhXUTBGQlFVRnBXWEUwWkZWSlFVRlBhak5xWW1neFVXZEJRVEJIWVZKMVNGWkRRVUZETkRGYVV6UmtWVWxCUVV0Q1JXMU1hREZSWjBGQmFVeFBZblZJVmtOQlFVSjNTWEFyTkdSVlNVRkJSbWxTYjNKb01WRm5RVUZSUVVOdGRVaFdRMEZCUVc5aU5tMDBaRlZKUVVGQ1JHVnlUR2d4VVdkQlFTdEZlWGQxU0ZaRFFVRkVaM1UzVHpSa1ZVbEJRVTFuY1hRM2FERlJaMEZCYzBwdE5uVklWa05CUVVOWlEwdzJOR1JWU1VGQlNVSXpkMkpvTVZGblFVRmhUMkpGZFVoV1EwRkJRbEZXWTJrMFpGVkpRVUZFYWtWNU4yZ3hVV2RCUVVsRVVGQjFTRlpEUVVGQlNXOTBTelJrVlVsQlFWQkJVVEZ5YURGUlowRkJNa2d2V25WSVZrTkJRVVJCTjNSNU5HUlZTVUZCUzJoa05FeG9NVkZuUVVGclRYcHFkVWhXUTBGQlFqUlBLMlUwWkZWSlFVRkhRM0UyY21neFVXZEJRVk5DYm5WMVNGWkRRVUZCZDJsUVJ6UmtWVWxCUVVKcU16bE1hREZSWjBGQlFVZGlOSFZJVmtOQlFVUnZNVkIxTkdSVlNVRkJUa0pFTHpkb01WRm5RVUYxVEVsRGRWaFdRMEZCUTJkSlVXRTFaRlZKUVVGSmFWRkRZbXd4VVdkQlFXTlFPRTExV0ZaRFFVRkNXV0pvUXpWa1ZVbEJRVVZFWkVVM2JERlJaMEZCUzBWM1dIVllWa05CUVVGUmRYaHhOV1JWU1VGQlVHZHdTSEpzTVZGblFVRTBTbWRvZFZoV1EwRkJSRWxDZVZjMVpGVkpRVUZNUWpKTFRHd3hVV2RCUVcxUFZYSjFXRlpEUVVGRFFWWkRLelZrVlVsQlFVZHFSRTF5YkRGUlowRkJWVVJKTW5WWVZrTkJRVUUwYjFSdE5XUlZTVUZCUTBGUlVHSnNNVkZuUVVGRFNEbEJkVmhXUTBGQlJIYzNWVTgxWkZWSlFVRk9hR05TTjJ3eFVXZEJRWGROZEV0MVdGWkRRVUZEYjA5ck5qVmtWVWxCUVVwRGNGVmliREZSWjBGQlpVSm9WblZZVmtOQlFVSm5hREZwTldSVlNVRkJSV295Vnpkc01WRm5RVUZOUjFabWRWaFdRMEZCUVZreFIwczFaRlZKUVVGQlFrUmFjbXd4VVdkQlFUWk1SbkIxV0ZaRFFVRkVVVWxITWpWa1ZVbEJRVXhwVUdOTWJERlJaMEZCYjFBMWVuVllWa05CUVVOSllsaGxOV1JWU1VGQlNFUmpaWEpzTVZGblFVRlhSWFFyZFZoV1F5SXNJbVIwZVhCbElqb2labXh2WVhRMk5DSXNJbk5vWVhCbElqcGJNVFEwWFgwc0lua2lPbnNpWDE5dVpHRnljbUY1WDE4aU9pSkJRVUZCUVZCWlkwZHJRVUZCUVVOQk15OXZXbEZCUVVGQlFVUktNa0pzUVVGQlFVRm5URXN5UjFWQlFVRkJRMmRWV210YVVVRkJRVUZQUkhkbGVHeEJRVUZCUVVGS1FtVkhWVUZCUVVGQ1FWbFZRVnBSUVVGQlFVdEJlVWxvYkVGQlFVRkJORUZOUlVkVlFVRkJRVVJuSzFCVldWRkJRVUZCUVVSMU5YaG9RVUZCUVVGQlQxQmFSMFZCUVVGQlEwRXlUVEJaVVVGQlFVRlBSRTUzVW1oQlFVRkJRVmxOVHpGSFJVRkJRVUZEWjI5UE9GbFJRVUZCUVVGQ0swdFNiRUZCUVVGQlVVWjBha2RWUVVGQlFVSm5lalJKV2xGQlFVRkJTVUpFYjJoc1FVRkJRVUZ2VEdaQ1IxVkJRVUZCUkdkbGNUUmFVVUZCUVVGQlFTdHRlR3hCUVVGQlFWRkJSMGxIVlVGQlFVRkNRV0V4WjFwUlFVRkJRVU5FVmt0Q2JFRkJRVUZCU1VRdk5VZEZRVUZCUVVSQllVdE5XVkZCUVVGQlJVTlRWRkpvUVVGQlFVRTBUSFl6UmpCQlFVRkJRMEZMZFVsWVVVRkJRVUZGUTFwNlFtUkJRVUZCUVRSQlpUTkdNRUZCUVVGQ1FTOWpRVmhSUVVGQlFVMUVlWGxvWkVGQlFVRkJTVTlxVlVZd1FVRkJRVUpuU1hWbldGRkJRVUZCVFVKakszaGtRVUZCUVVGQlNtTlBSMFZCUVVGQlJHZG9lbXRaVVVGQlFVRk5RalJhUW1oQlFVRkJRVzlIYlZCSFJVRkJRVUZEUVc1eE9GbFJRVUZCUVVsRVZIcDRhRUZCUVVGQldVRnFkMGRGUVVGQlFVSkJhazAwV1ZGQlFVRkJRVUZSY2xKb1FVRkJRVUUwU2s5TVIwVkJRVUZCUkdjMlZuZFpVVUZCUVVGTlFTOU1hR2hCUVVGQlFYZEtXQzlHTUVGQlFVRkJRVE5PTkZoUlFVRkJRVU5CYVhab1pFRkJRVUZCV1VkcFpFWXdRVUZCUVVGQk0xbHJXRkZCUVVGQlRVSlNaR2hrUVVGQlFVRlpUVnBwUmpCQlFVRkJRMEZMUm5OWVVVRkJRVUZMUTB0VmVHUkJRVUZCUVhkUGVFeEdNRUZCUVVGRFoydFhNRmhSUVVGQlFVZEJNbXA0WkVGQlFVRkJVVTUxZDBZd1FVRkJRVVJuSzNVNFdGRkJRVUZCU1VGaFRIaG9RVUZCUVVGSlJIQjFSMFZCUVVGQlJHZHBXRWxaVVVGQlFVRkxSRnBrYUdoQlFVRkJRVmxEYkRkSFJVRkJRVUZFWnk5VlkxbFJRVUZCUVVWRVUwWkNhRUZCUVVGQmQwdGlhRVl3UVVGQlFVTkJlazFqV0ZGQlFVRkJRMFI1Y2xKa1FVRkJRVUUwUW1WVlJqQkJRVUZCUTBGWU5GVllVVUZCUVVGRlEyNWthR1JCUVVGQlFUUlBOVzVHTUVGQlFVRkVRVkl6VlZoUlFVRkJRVXREWjJkb1pFRkJRVUZCWjFCdFVFWXdRVUZCUVVKblNVdFZXRkZCUVVGQlJVSklkV2hrUVVGQlFVRkpSemRRUmpCQlFVRkJRbWQwWmtsWVVVRkJRVUZOUkRoR1VtaEJRVUZCUVVGRlVUVkhSVUZCUVVGQlowRk1VVmxSUVVGQlFVTkRPRXhvYkVGQlFVRkJVVWhwY0VkVlFVRkJRVUpuTm1SeldsRkJRVUZCUjBKaFJHaHdRVUZCUVVGblRYUkJSMnRCUVVGQlFtZHhRVFJoVVVGQlFVRkRRMFl6UW14QlFVRkJRVUZIUzNGSFZVRkJRVUZEUVhGdWQxcFJRVUZCUVU5RWVWUm9iRUZCUVVGQldVUnphRWRWUVVGQlFVSm5NbEpOV2xGQlFVRkJTVUl6UW1oc1FVRkJRVUZuUWxnMVIwVkJRVUZCUVVGaVVVMWFVVUZCUVVGTFJFVkVVbXhCUVVGQlFVbENkMWxIVlVGQlFVRkNaMlJCUVZwUlFVRkJRVXRFVFRaQ2FFRkJRVUZCTkVOVVVrZEZRVUZCUVVOQk4yRnZXVkZCUVVGQlEwTXlhRUpvUVVGQlFVRjNTRFZsUjBWQlFVRkJRbWQxVlZWWlVVRkJRVUZQUkhwTVFtaEJRVUZCUVdkRE5GVkhSVUZCUVVGRFowRkJRVmxSUVVGQlFVOUVVelo0WkVGQlFVRkJRVXRZV0VZd1FVRkJRVU5uT1dOdldGRkJRVUZCUTBKSGRtaGtRVUZCUVVGM1NtRjRSakJCUVVGQlEwRjNZVGhZVVVGQlFVRkZSSE55VW1SQlFVRkJRVUZDWlhOR01FRkJRVUZFUVdKaGIxaFJRVUZCUVVsRVJYRkNaRUZCUVVGQlVVSjFia1l3UVVGQlFVTkJja3MwV0ZGQlFVRkJTMEU1ZEdoa1FVRkJRVUUwVFRZNVJqQkJRVUZCUkVGelQwRllVVUZCUVVGTlExTkJlR2hCUVVGQlFXOUlVVzFIUlVGQlFVRkVaM0F3U1ZsUlFVRkJRVVZFWWxob2FFRkJRVUZCWjBFMU4wZEZRVUZCUVVOQlpYQkJXVkZCUVVGQlIwUnRjRkpvUVVGQlFVRlpSa3MzUjBWQlFVRkJRbWRNVEd0WlVVRkJRVUZKUVVkMGVHaEJRVUZCUVdkUFF6QkhSVUZCUVVGRFFUUk1VVmxSUVVGQlFVbEVaM1JDYUVFaUxDSmtkSGx3WlNJNkltWnNiMkYwTmpRaUxDSnphR0Z3WlNJNld6RTBORjE5Zlgwc0ltbGtJam9pTURJd1pEVTBNRFF0T1dReU5TMDBNekJsTFRnMllXUXROamMyT1RZME9EWTBOR0kwSWl3aWRIbHdaU0k2SWtOdmJIVnRia1JoZEdGVGIzVnlZMlVpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWkdGMFlWOXpiM1Z5WTJVaU9uc2lhV1FpT2lKa1ltRTVPR0V3TkMwNU9EWmtMVFJrTW1ZdE9EZzJaaTFtT1RnMk1EYzVZakJtWWpjaUxDSjBlWEJsSWpvaVEyOXNkVzF1UkdGMFlWTnZkWEpqWlNKOUxDSm5iSGx3YUNJNmV5SnBaQ0k2SW1aa1lUZGpObU14TFdNek5qTXROREUzTWkwNE5HWXpMV0ppWXpOaFpHRmxNRFUyTVNJc0luUjVjR1VpT2lKTWFXNWxJbjBzSW1odmRtVnlYMmRzZVhCb0lqcHVkV3hzTENKdGRYUmxaRjluYkhsd2FDSTZiblZzYkN3aWJtOXVjMlZzWldOMGFXOXVYMmRzZVhCb0lqcDdJbWxrSWpvaU1tTTJZamM0WkRndE5XSXdZeTAwWldKaExXRXhaR0V0Wm1Rd1l6WTNOVGt6TWpjeElpd2lkSGx3WlNJNklreHBibVVpZlN3aWMyVnNaV04wYVc5dVgyZHNlWEJvSWpwdWRXeHNmU3dpYVdRaU9pSmxaR0V6WlRVeFppMHpOR1kxTFRSaE9HSXRPVFV4WkMxak1HSTJaVEF6TWpBNE1qa2lMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKa1lYUmhYM052ZFhKalpTSTZleUpwWkNJNkltUTVPRGcyTUdZeUxXTXdNemt0TkRnNU5DMWlPRFppTFdNMk9ETTFNRGMyTmpJMFl5SXNJblI1Y0dVaU9pSkRiMngxYlc1RVlYUmhVMjkxY21ObEluMHNJbWRzZVhCb0lqcDdJbWxrSWpvaVlqaGxOVGRtWWpBdFpqYzFNeTAwWmpNMExXSXhNV010T1Rrd1pXTmtOV0V3WXpWbElpd2lkSGx3WlNJNklreHBibVVpZlN3aWFHOTJaWEpmWjJ4NWNHZ2lPbTUxYkd3c0ltMTFkR1ZrWDJkc2VYQm9JanB1ZFd4c0xDSnViMjV6Wld4bFkzUnBiMjVmWjJ4NWNHZ2lPbnNpYVdRaU9pSTJaR1UwTXpNNU1TMWtZemhrTFRRM1l6SXRPVFZsWVMwME16STNaVFF6WkRnd016RWlMQ0owZVhCbElqb2lUR2x1WlNKOUxDSnpaV3hsWTNScGIyNWZaMng1Y0dnaU9tNTFiR3g5TENKcFpDSTZJakUyTmprMFpUZG1MVEF6WXpBdE5HWTNOaTFoWW1SaUxXTTFNVEk1TjJNMFpHUTVNQ0lzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltTmhiR3hpWVdOcklqcHVkV3hzTENKd2JHOTBJanA3SW1sa0lqb2lNV0ZqTjJZMlpURXROR1kzTWkwME5qSmlMV0prTTJJdE9USXpaR0k1WlRFME1UTTVJaXdpYzNWaWRIbHdaU0k2SWtacFozVnlaU0lzSW5SNWNHVWlPaUpRYkc5MEluMHNJbkpsYm1SbGNtVnljeUk2VzNzaWFXUWlPaUpsWkdFelpUVXhaaTB6TkdZMUxUUmhPR0l0T1RVeFpDMWpNR0kyWlRBek1qQTRNamtpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlYU3dpZEc5dmJIUnBjSE1pT2x0YklrNWhiV1VpTENKSE1WOVRVMVJmUjB4UFFrRk1JbDBzV3lKQ2FXRnpJaXdpTUM0NE1DSmRMRnNpVTJ0cGJHd2lMQ0l4TGpBNUlsMWRmU3dpYVdRaU9pSTNOamM1WXpnME9DMWpNRGMxTFRSalkySXRPREJsTXkwMk4yTTRaV1ZqT1RReVlqY2lMQ0owZVhCbElqb2lTRzkyWlhKVWIyOXNJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhwYm1WZllXeHdhR0VpT25zaWRtRnNkV1VpT2pBdU1YMHNJbXhwYm1WZlkyRndJam9pY205MWJtUWlMQ0pzYVc1bFgyTnZiRzl5SWpwN0luWmhiSFZsSWpvaUl6Rm1OemRpTkNKOUxDSnNhVzVsWDJwdmFXNGlPaUp5YjNWdVpDSXNJbXhwYm1WZmQybGtkR2dpT25zaWRtRnNkV1VpT2pWOUxDSjRJanA3SW1acFpXeGtJam9pZUNKOUxDSjVJanA3SW1acFpXeGtJam9pZVNKOWZTd2lhV1FpT2lJMlpHVTBNek01TVMxa1l6aGtMVFEzWXpJdE9UVmxZUzAwTXpJM1pUUXpaRGd3TXpFaUxDSjBlWEJsSWpvaVRHbHVaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzYVc1bFgyTmhjQ0k2SW5KdmRXNWtJaXdpYkdsdVpWOWpiMnh2Y2lJNmV5SjJZV3gxWlNJNklpTm1aV1V3T0dJaWZTd2liR2x1WlY5cWIybHVJam9pY205MWJtUWlMQ0pzYVc1bFgzZHBaSFJvSWpwN0luWmhiSFZsSWpvMWZTd2llQ0k2ZXlKbWFXVnNaQ0k2SW5naWZTd2llU0k2ZXlKbWFXVnNaQ0k2SW5raWZYMHNJbWxrSWpvaVlqaGxOVGRtWWpBdFpqYzFNeTAwWmpNMExXSXhNV010T1Rrd1pXTmtOV0V3WXpWbElpd2lkSGx3WlNJNklreHBibVVpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWTJGc2JHSmhZMnNpT201MWJHd3NJbU52YkhWdGJsOXVZVzFsY3lJNld5SjRJaXdpZVNKZExDSmtZWFJoSWpwN0luZ2lPbnNpWDE5dVpHRnljbUY1WDE4aU9pSkJRVUZ2S3k5SE1tUlZTVUZCUWtKeE9XSmFNVkZuUVVFclRtbzBkRzVXUTBGQlJHZFNMM2t5WkZWSlFVRk5hVEl2TjFveFVXZEJRWE5EVlVSME0xWkRRVUZEV1d4QllUTmtWVWxCUVVsQlJFTnlaREZSWjBGQllVaEpUblF6VmtOQlFVSlJORkpETTJSVlNVRkJSR2hSUmt4a01WRm5RVUZKVERoWWRETldRMEZCUVVsTWFIVXpaRlZKUVVGUVEyTkljbVF4VVdkQlFUSkJjMmwwTTFaRFFVRkVRV1ZwVnpOa1ZVbEJRVXRxY0V0TVpERlJaMEZCYTBabmMzUXpWa05CUVVJMGVIa3JNMlJWU1VGQlIwRXlUVGRrTVZGblFVRlRTMVV5ZEROV1EwRkJRWGRHUkhFelpGVkpRVUZDYVVSUVltUXhVV2RCUVVGUVNrRjBNMVpEUVVGRWIxbEZVek5rVlVsQlFVNUVVRkkzWkRGUlowRkJkVVExVEhRelZrTkJRVU5uY2xVMk0yUlZTVUZCU1dkalZYSmtNVkZuUVVGalNYUldkRE5XUTBGQlFsa3JiR2t6WkZWSlFVRkZRbkJZVEdReFVXZEJRVXRPYUdaME0xWkRRVUZCVVZJeVR6TmtWVWxCUVZCcE1WcHlaREZSWjBGQk5FTlNjWFF6VmtOQlFVUkphekl5TTJSVlNVRkJURUZEWTJKa01WRm5RVUZ0U0VZd2RETldRMEZCUTBFMFNHVXpaRlZKUVVGSGFGQmxOMlF4VVdkQlFWVk1OU3QwTTFaRFFVRkJORXhaU3pOa1ZVbEJRVU5EWTJoaVpERlJaMEZCUTBGMVNuUXpWa05CUVVSM1pWbDVNMlJWU1VGQlRtcHZhamRrTVZGblFVRjNSbVZVZEROV1EwRkJRMjk0Y0dFelpGVkpRVUZLUVRGdGNtUXhVV2RCUVdWTFUyUjBNMVpEUVVGQ1owVTJSek5rVlVsQlFVVnBRM0JNWkRGUlowRkJUVkJIYm5RelZrTkJRVUZaV1V0MU0yUlZTVUZCUVVSUWNuSmtNVkZuUVVFMlJESjVkRE5XUTBGQlJGRnlURmN6WkZWSlFVRk1aMkoxWW1ReFVXZEJRVzlKY1RoME0xWkRRVUZEU1N0aUt6TmtWVWxCUVVoQ2IzYzNaREZSWjBGQlYwNW1SM1F6VmtOQlFVSkJVbk54TTJSVlNVRkJRMmt4ZW1Ka01WRm5RVUZGUTFSU2RETldRMEZCUkRScmRGTXpaRlZKUVVGUFFVSXlUR1F4VVdkQlFYbElSR0owTTFaRFFVRkRkek01TmpOa1ZVbEJRVXBvVHpSeVpERlJaMEZCWjB3emJIUXpWa05CUVVKdlRFOXRNMlJWU1VGQlJrTmlOMHhrTVZGblFVRlBRWEozZEROV1EwRkJRV2RsWms4elpGVkpRVUZCYW04NWNtUXhVV2RCUVRoR1lqWjBNMVpEUVVGRVdYaG1Nak5rVlVsQlFVMUJNRUZpYURGUlowRkJjVXROUlhWSVZrTkJRVU5SUldkcE5HUlZTVUZCU0dsQ1F6ZG9NVkZuUVVGWlVFRlBkVWhXUTBGQlFrbFllRXMwWkZWSlFVRkVSRTlHWW1neFVXZEJRVWRFTUZwMVNGWkRRVUZCUVhKQ2VUUmtWVWxCUVU5bllVbE1hREZSWjBGQk1FbHJhblZJVmtOQlFVTTBLME5oTkdSVlNVRkJTMEp1UzNKb01WRm5RVUZwVGxsMGRVaFdRMEZCUW5kU1ZFYzBaRlZKUVVGR2FUQk9UR2d4VVdkQlFWRkRUVFIxU0ZaRFFVRkJiMnRxZFRSa1ZVbEJRVUpCUWxBM2FERlJaMEZCSzBjNVEzVklWa05CUVVSbk0ydFhOR1JWU1VGQlRXaE9VMkpvTVZGblFVRnpUSGhOZFVoV1EwRkJRMWxMTVVNMFpGVkpRVUZKUTJGVk4yZ3hVV2RCUVdGQmJGaDFTRlpEUVVGQ1VXVkdjVFJrVlVsQlFVUnFibGhpYURGUlowRkJTVVphYUhWSVZrTkJRVUZKZUZkVE5HUlZTVUZCVUVGNllVeG9NVkZuUVVFeVMwcHlkVWhXUTBGQlJFRkZWeXMwWkZWSlFVRkxhVUZqY21neFVXZEJRV3RQT1RGMVNGWkRRVUZDTkZodWJUUmtWVWxCUVVkRVRtWk1hREZSWjBGQlUwUjVRWFZJVmtOQlFVRjNjVFJQTkdSVlNVRkJRbWRoYURkb01WRm5RVUZCU1cxTGRVaFdRMEZCUkc4NU5ESTBaRlZKUVVGT1FtMXJZbWd4VVdkQlFYVk9WMVYxU0ZaRFFVRkRaMUpLYVRSa1ZVbEJRVWxwZW0wM2FERlJaMEZCWTBOTFpuVklWa05CUVVKWmEyRkxOR1JWU1VGQlJVRkJjSEpvTVZGblFVRkxSeXR3ZFVoV1EwRkJRVkV6Y1hrMFpGVkpRVUZRYUUxelRHZ3hVV2RCUVRSTWRYcDFTRlpEUVVGRVNVdHlaVFJrVlVsQlFVeERXblZ5YURGUlowRkJiVUZwSzNWSVZrTkJRVU5CWkRoSE5HUlZTVUZCUjJwdGVFeG9NVkZuUFQwaUxDSmtkSGx3WlNJNkltWnNiMkYwTmpRaUxDSnphR0Z3WlNJNld6RXpOMTE5TENKNUlqcDdJbDlmYm1SaGNuSmhlVjlmSWpvaVFVRkJRVUZCUVVGSWEwRkJRVUZCUVVGQlFXVlJSRTE2VFhwTmVrMTRNVUZhYlZwdFdtMWFiVWhGUVVGQlFVRkJRVUZCWTFGRVRYcE5lazE2VFhoMFFYcGplazE2VFhwTlIydENiVnB0V20xYWJWbGhVVWRhYlZwdFdtMWFhSEJCZW1ONlRYcE5lazFIYTBST2VrMTZUWHBOZDJGUlRUTk5lazE2VFhwQ2NFRjZZM3BOZWsxNlRVaEZSRTU2VFhwTmVrMTNZMUZCUVVGQlFVRkJRVUkxUVcxd2JWcHRXbTFhU0RCQlFVRkJRVUZCUVVGbFVVRkJRVUZCUVVGQlFqVkJUWHBOZWsxNlRYcElWVVJPZWsxNlRYcE5kMk5SUVVGQlFVRkJRVUZDZUVGNlkzcE5lazE2VFVkclFVRkJRVUZCUVVGQllWRktjVnB0V20xYWJWSnNRVTE2VFhwTmVrMTZSekJCZWsxNlRYcE5lazFpVVVSTmVrMTZUWHBOZUhSQlRYcE5lazE2VFhwSE1FRjZUWHBOZWsxNlRXSlJUVE5OZWsxNlRYcENjRUZOZWsxNlRYcE5la2N3UVhwTmVrMTZUWHBOWWxGRVRYcE5lazE2VFhoMFFVMTZUWHBOZWsxNlJ6QkJlazE2VFhwTmVrMWlVVXB4V20xYWJWcHRVblJCUVVGQlFVRkJRVUZJUlVGNlRYcE5lazE2VFdSUlNuRmFiVnB0V20xU01VRjZZM3BOZWsxNlRVaHJRbTFhYlZwdFdtMVpaMUZFVFhwTmVrMTZUWGxDUVUxNlRYcE5lazE2U1VWQ2JWcHRXbTFhYlZsblVVcHhXbTFhYlZwdFUwSkJXbTFhYlZwdFdtMUpSVUZCUVVGQlFVRkJRV2RSU25GYWJWcHRXbTFTT1VGNlkzcE5lazE2VFVoclJFNTZUWHBOZWsxM1pWRkVUWHBOZWsxNlRYZzVRVnB0V20xYWJWcHRTR3RDYlZwdFdtMWFiVmxsVVVkYWJWcHRXbTFhYURWQldtMWFiVnB0V20xSWEwRkJRVUZCUVVGQlFXVlJRVUZCUVVGQlFVRkNOVUZCUVVGQlFVRkJRVWhyUTJGdFdtMWFiVnByWkZGS2NWcHRXbTFhYlZJeFFVRkJRVUZCUVVGQlNHdEVUbnBOZWsxNlRYZGxVVUZCUVVGQlFVRkJRMEpCUVVGQlFVRkJRVUZKUlVGQlFVRkJRVUZCUVdkUlFVRkJRVUZCUVVGRFFrRmFiVnB0V20xYWJVbEZRMkZ0V20xYWJWcHJabEZLY1ZwdFdtMWFiVkk1UVcxd2JWcHRXbTFhU0RCQlFVRkJRVUZCUVVGblVVUk5lazE2VFhwTmVVSkJRVUZCUVVGQlFVRkpSVU5oYlZwdFdtMWFhMlpSUkUxNlRYcE5lazE0T1VGdGNHMWFiVnB0V2tnd1FVRkJRVUZCUVVGQloxRkhXbTFhYlZwdFdtZzFRVnB0V20xYWJWcHRTR3RDYlZwdFdtMWFiVmxsVVUwelRYcE5lazE2UWpWQmVtTjZUWHBOZWsxSWEwSnRXbTFhYlZwdFdXVlJSMXB0V20xYWJWcG9OVUZhYlZwdFdtMWFiVWhyUW0xYWJWcHRXbTFaWlZGSFdtMWFiVnB0V21nMVFWcHRXbTFhYlZwdFNHdENiVnB0V20xYWJWbGxVVTB6VFhwTmVrMTZRalZCV20xYWJWcHRXbTFKUlVKdFdtMWFiVnB0V1dkUlIxcHRXbTFhYlZwcFFrRmFiVnB0V20xYWJVbEZRMkZ0V20xYWJWcHJaMUZFVFhwTmVrMTZUWGxDUVVGQlFVRkJRVUZCU1VWRFlXMWFiVnB0V210bVVVMHpUWHBOZWsxNlFqVkJlbU42VFhwTmVrMUlhMEp0V20xYWJWcHRXV1ZSUjFwdFdtMWFiVnBvTlVGYWJWcHRXbTFhYlVoclFtMWFiVnB0V20xWlpWRkhXbTFhYlZwdFdtZzFRVnB0V20xYWJWcHRTR3RDYlZwdFdtMWFiVmxsVVVkYWJWcHRXbTFhYURWQldtMWFiVnB0V20xSWEwRjZUWHBOZWsxNlRXWlJSRTE2VFhwTmVrMTRPVUZ0Y0cxYWJWcHRXa2d3UTJGdFdtMWFiVnByWmxGS2NWcHRXbTFhYlZJNVFVMTZUWHBOZWsxNlNEQkVUbnBOZWsxNlRYZGxVVVJOZWsxNlRYcE5lREZCUVVGQlFVRkJRVUZJYTBGNlRYcE5lazE2VFdSUlJFMTZUWHBOZWsxNE1VRk5lazE2VFhwTmVraFZRMkZ0V20xYWJWcHJaRkZLY1ZwdFdtMWFiVkl4UVcxd2JWcHRXbTFhU0ZWQmVrMTZUWHBOZWsxa1VVUk5lazE2VFhwTmVERkJUWHBOZWsxNlRYcElWVVJPZWsxNlRYcE5kMk5SUjFwdFdtMWFiVnBvZUVGYWJWcHRXbTFhYlVoRlFVRkJRVUZCUVVGQlkxRkJRVUZCUVVGQlFVSjRRVUZCUVVGQlFVRkJTRVZCZWsxNlRYcE5lazFrVVVweFdtMWFiVnB0VWpsQlFVRkJRVUZCUVVGSlZVUk9lazE2VFhwTmQyZFJRVDA5SWl3aVpIUjVjR1VpT2lKbWJHOWhkRFkwSWl3aWMyaGhjR1VpT2xzeE16ZGRmWDE5TENKcFpDSTZJbVE1T0RnMk1HWXlMV013TXprdE5EZzVOQzFpT0RaaUxXTTJPRE0xTURjMk5qSTBZeUlzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhoWW1Wc0lqcDdJblpoYkhWbElqb2lTRmxEVDAwaWZTd2ljbVZ1WkdWeVpYSnpJanBiZXlKcFpDSTZJbUl6WVRVME1UVmhMV0l5TjJZdE5HRTNaUzFpTXpoaExUSmtOMlF4TkdKak1XWTJPQ0lzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMWRmU3dpYVdRaU9pSmlNR1F5T1dGaVlTMHlZV1V5TFRReU5UVXRPREZpTVMwME5UVTFOekEzTlRobE5EVWlMQ0owZVhCbElqb2lUR1ZuWlc1a1NYUmxiU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpqWVd4c1ltRmpheUk2Ym5Wc2JDd2lZMjlzZFcxdVgyNWhiV1Z6SWpwYkluZ2lMQ0o1SWwwc0ltUmhkR0VpT25zaWVDSTZleUpmWDI1a1lYSnlZWGxmWHlJNklrRkJRa0ZxVHpZeVpGVkpRVUZEYWpjNFlsb3hVV2RCUVVWSGNqRjBibFpEUVVGQlFUaHJRek5rVlVsQlFVOW9aMUpNWkRGUlowRkJNRTA1U0hRelZrTkJRVVJCVmpWUE0yUlZTVUZCUzJwSGJISmtNVkZuUVVGclJGZGhkRE5XUTBGQlEwRjJaVmN6WkZWSlFVRkhaM00yWW1ReFVXZEJRVlZLZG5OME0xWkRRVUZDUVVsNmFUUmtWVWxCUVVOcFUwODNhREZSWjBGQlJVRkZMM1ZJVmtOQlFVRkJhVmx4TkdSVlNVRkJUMm96YW1Kb01WRm5RVUV3UjJGU2RVaFdRMEZCUkVFM2RIazBaRlZKUVVGTGFHUTBUR2d4VVdkQlFXdE5lbXAxU0ZaRFFVRkRRVlpES3pWa1ZVbEJRVWRxUkUxeWJERlJaMEZCVlVSSk1uVllWa05CUVVKQmRXOUhOV1JWU1VGQlEyZHdhR0pzTVZGblFVRkZTbWxKZFZoV1F5SXNJbVIwZVhCbElqb2labXh2WVhRMk5DSXNJbk5vWVhCbElqcGJNamRkZlN3aWVTSTZleUpmWDI1a1lYSnlZWGxmWHlJNklrRkJRVUZaUTJwSFJ6QkJRVUZCUVdjMk4zZGlVVUZCUVVGTlEzUnplSFJCUVVGQlFWbEhibTlIYTBGQlFVRkVaMk5tT0dGUlFVRkJRVWRDTmtab2RFRkJRVUZCTkVSVlVraFZRVUZCUVVSQmFYbHpaRkZCUVVGQlNVUm9VbEl4UVVGQlFVRjNSVWRLU0RCQlFVRkJSR2RTTTBWbVVVRkJRVUZCUWs5WFVqbEJRVUZCUVZsT1VrcElWVUZCUVVGRVFYa3dSV1JSUVVGQlFVTkVSRTlTTVVGQlFVRkJXVUZYU2toRlFVRkJRVUZuY2pSdlkxRkJRVUZCVDBKWmFrSjRRVUZCUVVGM1R6WjNTRVZCUVVGQlFVRkxZa0ZqVVVGQlFVRkRRbXB5ZUhoQlFVRkJRWGRIVDJWSVJVRkJRVUZCWjJwaFNXTlJRVUZCUVVsRE1uQm9lRUZCUVVGQmIwVlJRMGhWUVVGQlFVTm5Va0ZKWkZGQlFVRkJTMEpGUVdneFFTSXNJbVIwZVhCbElqb2labXh2WVhRMk5DSXNJbk5vWVhCbElqcGJNamRkZlgxOUxDSnBaQ0k2SW1SbVlUTTNNR1U0TFdWbFlqUXROREUwTUMxaVlqTXpMVGd4WldGaU1EUTBZVEk1WWlJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14aFltVnNJanA3SW5aaGJIVmxJam9pVDJKelpYSjJZWFJwYjI1ekluMHNJbkpsYm1SbGNtVnljeUk2VzNzaWFXUWlPaUl4TmpZNU5HVTNaaTB3TTJNd0xUUm1Oell0WVdKa1lpMWpOVEV5T1Rkak5HUmtPVEFpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlYWDBzSW1sa0lqb2lOemRtTkRKaE5UVXRZamswWVMwMFpEazBMVGd4T1dNdE1UQmtPRGt6TnpSak1XUTNJaXdpZEhsd1pTSTZJa3hsWjJWdVpFbDBaVzBpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdsdVpWOWhiSEJvWVNJNmV5SjJZV3gxWlNJNk1DNHhmU3dpYkdsdVpWOWpZWEFpT2lKeWIzVnVaQ0lzSW14cGJtVmZZMjlzYjNJaU9uc2lkbUZzZFdVaU9pSWpNV1kzTjJJMEluMHNJbXhwYm1WZmFtOXBiaUk2SW5KdmRXNWtJaXdpYkdsdVpWOTNhV1IwYUNJNmV5SjJZV3gxWlNJNk5YMHNJbmdpT25zaVptbGxiR1FpT2lKNEluMHNJbmtpT25zaVptbGxiR1FpT2lKNUluMTlMQ0pwWkNJNklqRXpOR0ZrT1dOakxUTTJaV0V0TkdWaU15MDVPRE16TFdNNFpUQTJNR0V3WVdFd1l5SXNJblI1Y0dVaU9pSk1hVzVsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14cGJtVmZZMkZ3SWpvaWNtOTFibVFpTENKc2FXNWxYMk52Ykc5eUlqcDdJblpoYkhWbElqb2lJMlExTTJVMFppSjlMQ0pzYVc1bFgycHZhVzRpT2lKeWIzVnVaQ0lzSW14cGJtVmZkMmxrZEdnaU9uc2lkbUZzZFdVaU9qVjlMQ0o0SWpwN0ltWnBaV3hrSWpvaWVDSjlMQ0o1SWpwN0ltWnBaV3hrSWpvaWVTSjlmU3dpYVdRaU9pSmxZekF4TjJRMFlTMDRORGMyTFRSbU5HWXRZV0l5T1MwMU1ESm1ZMlZqWm1SbE9Ua2lMQ0owZVhCbElqb2lUR2x1WlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKdGIyNTBhSE1pT2xzd0xEWmRmU3dpYVdRaU9pSTBabVpoT1RSaU9DMDBNV001TFRReU1qTXRZamhrWXkwd01UTTVPVFkxTkdGbFkySWlMQ0owZVhCbElqb2lUVzl1ZEdoelZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14cGJtVmZZMkZ3SWpvaWNtOTFibVFpTENKc2FXNWxYMk52Ykc5eUlqcDdJblpoYkhWbElqb2lJems1WkRVNU5DSjlMQ0pzYVc1bFgycHZhVzRpT2lKeWIzVnVaQ0lzSW14cGJtVmZkMmxrZEdnaU9uc2lkbUZzZFdVaU9qVjlMQ0o0SWpwN0ltWnBaV3hrSWpvaWVDSjlMQ0o1SWpwN0ltWnBaV3hrSWpvaWVTSjlmU3dpYVdRaU9pSm1NbUppTnpRNFpTMDBNREk1TFRSa1l6RXRPVGs0TXkweU9ESTVZMkpqTVRnNE1URWlMQ0owZVhCbElqb2lUR2x1WlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKdGIyNTBhSE1pT2xzd0xEUXNPRjE5TENKcFpDSTZJalUyTURoaE1XUTRMV0ZrTVRndE5EVXlNUzFoT0RobExXVTVNRFprT1RFelpXSmhOaUlzSW5SNWNHVWlPaUpOYjI1MGFITlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVpHRjBZVjl6YjNWeVkyVWlPbnNpYVdRaU9pSXdNakJrTlRRd05DMDVaREkxTFRRek1HVXRPRFpoWkMwMk56WTVOalE0TmpRMFlqUWlMQ0owZVhCbElqb2lRMjlzZFcxdVJHRjBZVk52ZFhKalpTSjlMQ0puYkhsd2FDSTZleUpwWkNJNkltWXlZbUkzTkRobExUUXdNamt0TkdSak1TMDVPVGd6TFRJNE1qbGpZbU14T0RneE1TSXNJblI1Y0dVaU9pSk1hVzVsSW4wc0ltaHZkbVZ5WDJkc2VYQm9JanB1ZFd4c0xDSnRkWFJsWkY5bmJIbHdhQ0k2Ym5Wc2JDd2libTl1YzJWc1pXTjBhVzl1WDJkc2VYQm9JanA3SW1sa0lqb2lNemRpT1RGbE9EVXRNVEZpWkMwME5qazRMV0l5TmpFdE56Sm1NakUzTlRnek5UbGtJaXdpZEhsd1pTSTZJa3hwYm1VaWZTd2ljMlZzWldOMGFXOXVYMmRzZVhCb0lqcHVkV3hzZlN3aWFXUWlPaUkwTURJM1pEYzFOaTA0Tnpjd0xUUmhNelF0WVdSa05TMWlNRFV6WVdZek1HSXhObVVpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnRiMjUwYUhNaU9sc3dMRElzTkN3MkxEZ3NNVEJkZlN3aWFXUWlPaUpqTldNek1qYzBNeTFrTVRnekxUUmpZemd0T0RVM015MW1ZemsxTVRBNU4yUXpaaklpTENKMGVYQmxJam9pVFc5dWRHaHpWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltbDBaVzF6SWpwYmV5SnBaQ0k2SWpreVl6Tm1aVEZtTFRWa05HWXROREZsTmkxaE1ETXlMV1F5Tm1FMll6VTBZV0V3TWlJc0luUjVjR1VpT2lKTVpXZGxibVJKZEdWdEluMHNleUpwWkNJNklqQXpOamhoT1dJekxURmxZbVV0TkRWaFpTMWhZelpqTFRFME5qVmpPR016WlRaaE5TSXNJblI1Y0dVaU9pSk1aV2RsYm1SSmRHVnRJbjBzZXlKcFpDSTZJamsxT0RjMk1HUm1MV0UxTVRRdE5EazFOaTFoT0RjMUxXRTBPREJsTUdJNFptVmlZaUlzSW5SNWNHVWlPaUpNWldkbGJtUkpkR1Z0SW4wc2V5SnBaQ0k2SWpjM1pqUXlZVFUxTFdJNU5HRXROR1E1TkMwNE1UbGpMVEV3WkRnNU16YzBZekZrTnlJc0luUjVjR1VpT2lKTVpXZGxibVJKZEdWdEluMHNleUpwWkNJNklqSXpPVEl5TXpjd0xUazVOREF0TkRBNFlTMWhaVFk0TFRFMVptRXlZMk5tWTJVNVlTSXNJblI1Y0dVaU9pSk1aV2RsYm1SSmRHVnRJbjBzZXlKcFpDSTZJbUl3WkRJNVlXSmhMVEpoWlRJdE5ESTFOUzA0TVdJeExUUTFOVFUzTURjMU9HVTBOU0lzSW5SNWNHVWlPaUpNWldkbGJtUkpkR1Z0SW4xZExDSndiRzkwSWpwN0ltbGtJam9pTVdGak4yWTJaVEV0TkdZM01pMDBOakppTFdKa00ySXRPVEl6WkdJNVpURTBNVE01SWl3aWMzVmlkSGx3WlNJNklrWnBaM1Z5WlNJc0luUjVjR1VpT2lKUWJHOTBJbjE5TENKcFpDSTZJbVF4TW1NeVpEWXpMV1F5TXpVdE5EQTJPUzA1TldWa0xURTRZelV6WTJWak0yVTVOQ0lzSW5SNWNHVWlPaUpNWldkbGJtUWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liR2x1WlY5aGJIQm9ZU0k2ZXlKMllXeDFaU0k2TUM0eGZTd2liR2x1WlY5allYQWlPaUp5YjNWdVpDSXNJbXhwYm1WZlkyOXNiM0lpT25zaWRtRnNkV1VpT2lJak1XWTNOMkkwSW4wc0lteHBibVZmYW05cGJpSTZJbkp2ZFc1a0lpd2liR2x1WlY5M2FXUjBhQ0k2ZXlKMllXeDFaU0k2Tlgwc0luZ2lPbnNpWm1sbGJHUWlPaUo0SW4wc0lua2lPbnNpWm1sbGJHUWlPaUo1SW4xOUxDSnBaQ0k2SWpNM1lqa3haVGcxTFRFeFltUXRORFk1T0MxaU1qWXhMVGN5WmpJeE56VTRNelU1WkNJc0luUjVjR1VpT2lKTWFXNWxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhoWW1Wc0lqcDdJblpoYkhWbElqb2lSekZmVTFOVVgwZE1UMEpCVENKOUxDSnlaVzVrWlhKbGNuTWlPbHQ3SW1sa0lqb2laV1JoTTJVMU1XWXRNelJtTlMwMFlUaGlMVGsxTVdRdFl6QmlObVV3TXpJd09ESTVJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZWMTlMQ0pwWkNJNklqa3lZek5tWlRGbUxUVmtOR1l0TkRGbE5pMWhNRE15TFdReU5tRTJZelUwWVdFd01pSXNJblI1Y0dVaU9pSk1aV2RsYm1SSmRHVnRJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhoWW1Wc0lqcDdJblpoYkhWbElqb2lUa1ZEVDBaVFgwMWhjM05DWVhraWZTd2ljbVZ1WkdWeVpYSnpJanBiZXlKcFpDSTZJalF3TWpka056VTJMVGczTnpBdE5HRXpOQzFoWkdRMUxXSXdOVE5oWmpNd1lqRTJaU0lzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMWRmU3dpYVdRaU9pSXdNelk0WVRsaU15MHhaV0psTFRRMVlXVXRZV00yWXkweE5EWTFZemhqTTJVMllUVWlMQ0owZVhCbElqb2lUR1ZuWlc1a1NYUmxiU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpqWVd4c1ltRmpheUk2Ym5Wc2JDd2ljR3h2ZENJNmV5SnBaQ0k2SWpGaFl6ZG1ObVV4TFRSbU56SXRORFl5WWkxaVpETmlMVGt5TTJSaU9XVXhOREV6T1NJc0luTjFZblI1Y0dVaU9pSkdhV2QxY21VaUxDSjBlWEJsSWpvaVVHeHZkQ0o5TENKeVpXNWtaWEpsY25NaU9sdDdJbWxrSWpvaU5UZzRPRE0xTm1RdE16Y3pPUzAwTWpOakxUbGtZak10WmpjNU9EVTJOelExWWpreElpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlYwc0luUnZiMngwYVhCeklqcGJXeUpPWVcxbElpd2lVMFZEVDA5U1FWOU9RMU5WWDBOT1FWQlRJbDBzV3lKQ2FXRnpJaXdpTFRBdU9EUWlYU3hiSWxOcmFXeHNJaXdpTUM0ME55SmRYWDBzSW1sa0lqb2lZbVEwTlRsbFl6WXROV05qTmkwME5Ua3hMVGhrTWprdFpqSXlZMlkxTnpFd05qWmpJaXdpZEhsd1pTSTZJa2h2ZG1WeVZHOXZiQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzWVdKbGJDSTZleUoyWVd4MVpTSTZJbE5GUTA5UFVrRmZUa05UVlY5RFRrRlFVeUo5TENKeVpXNWtaWEpsY25NaU9sdDdJbWxrSWpvaU5UZzRPRE0xTm1RdE16Y3pPUzAwTWpOakxUbGtZak10WmpjNU9EVTJOelExWWpreElpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlYxOUxDSnBaQ0k2SWpJek9USXlNemN3TFRrNU5EQXROREE0WVMxaFpUWTRMVEUxWm1FeVkyTm1ZMlU1WVNJc0luUjVjR1VpT2lKTVpXZGxibVJKZEdWdEluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltTmhiR3hpWVdOcklqcHVkV3hzTENKd2JHOTBJanA3SW1sa0lqb2lNV0ZqTjJZMlpURXROR1kzTWkwME5qSmlMV0prTTJJdE9USXpaR0k1WlRFME1UTTVJaXdpYzNWaWRIbHdaU0k2SWtacFozVnlaU0lzSW5SNWNHVWlPaUpRYkc5MEluMHNJbkpsYm1SbGNtVnljeUk2VzNzaWFXUWlPaUl4TmpZNU5HVTNaaTB3TTJNd0xUUm1Oell0WVdKa1lpMWpOVEV5T1Rkak5HUmtPVEFpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlYU3dpZEc5dmJIUnBjSE1pT2x0YklrNWhiV1VpTENKUFFsTmZSRUZVUVNKZExGc2lRbWxoY3lJc0lrNUJJbDBzV3lKVGEybHNiQ0lzSWs1QklsMWRmU3dpYVdRaU9pSTNaVFU0TTJVeVl5MWxNelZoTFRRME1ETXRZakF3T1MxaVl6QXpZbVk1TmpZeE1ETWlMQ0owZVhCbElqb2lTRzkyWlhKVWIyOXNJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbVJoZVhNaU9sc3hMRGdzTVRVc01qSmRmU3dpYVdRaU9pSTFZVGhoWldRNU5TMHlNVFUxTFRRd05XWXRZVGs0TUMwMk5HSmlOR1UyWmpjek1qa2lMQ0owZVhCbElqb2lSR0Y1YzFScFkydGxjaUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUp0YjI1MGFITWlPbHN3TERFc01pd3pMRFFzTlN3MkxEY3NPQ3c1TERFd0xERXhYWDBzSW1sa0lqb2lPR1k0WlRjME5UWXRPRFJqTmkwME1tUmhMV0U0TWpndFpHTTBNakV3WldFeU9ERmlJaXdpZEhsd1pTSTZJazF2Ym5Sb2MxUnBZMnRsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKd2JHOTBJanA3SW1sa0lqb2lNV0ZqTjJZMlpURXROR1kzTWkwME5qSmlMV0prTTJJdE9USXpaR0k1WlRFME1UTTVJaXdpYzNWaWRIbHdaU0k2SWtacFozVnlaU0lzSW5SNWNHVWlPaUpRYkc5MEluMTlMQ0pwWkNJNklqY3lOalkyTkROaUxUVXpOR1F0TkRobVlTMWhPRFkwTFRKaU1ESmtaR1JsTkRaall5SXNJblI1Y0dVaU9pSlNaWE5sZEZSdmIyd2lmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liM1psY214aGVTSTZleUpwWkNJNklqUTFOVFEyWmpFM0xXWmhOakF0TkRZNVl5MDROakl4TFdNeVlqSTNOVEl6TVdNek15SXNJblI1Y0dVaU9pSkNiM2hCYm01dmRHRjBhVzl1SW4wc0luQnNiM1FpT25zaWFXUWlPaUl4WVdNM1pqWmxNUzAwWmpjeUxUUTJNbUl0WW1RellpMDVNak5rWWpsbE1UUXhNemtpTENKemRXSjBlWEJsSWpvaVJtbG5kWEpsSWl3aWRIbHdaU0k2SWxCc2IzUWlmWDBzSW1sa0lqb2lNbUUzTldObVpUY3RZV000T0MwMFpHVTJMV0k0TW1ZdE1qUTJaamswT0dVM09HSTBJaXdpZEhsd1pTSTZJa0p2ZUZwdmIyMVViMjlzSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14cGJtVmZZV3h3YUdFaU9uc2lkbUZzZFdVaU9qQXVNWDBzSW14cGJtVmZZMkZ3SWpvaWNtOTFibVFpTENKc2FXNWxYMk52Ykc5eUlqcDdJblpoYkhWbElqb2lJekZtTnpkaU5DSjlMQ0pzYVc1bFgycHZhVzRpT2lKeWIzVnVaQ0lzSW14cGJtVmZkMmxrZEdnaU9uc2lkbUZzZFdVaU9qVjlMQ0o0SWpwN0ltWnBaV3hrSWpvaWVDSjlMQ0o1SWpwN0ltWnBaV3hrSWpvaWVTSjlmU3dpYVdRaU9pSXlZelppTnpoa09DMDFZakJqTFRSbFltRXRZVEZrWVMxbVpEQmpOamMxT1RNeU56RWlMQ0owZVhCbElqb2lUR2x1WlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKaWIzUjBiMjFmZFc1cGRITWlPaUp6WTNKbFpXNGlMQ0ptYVd4c1gyRnNjR2hoSWpwN0luWmhiSFZsSWpvd0xqVjlMQ0ptYVd4c1gyTnZiRzl5SWpwN0luWmhiSFZsSWpvaWJHbG5hSFJuY21WNUluMHNJbXhsWm5SZmRXNXBkSE1pT2lKelkzSmxaVzRpTENKc1pYWmxiQ0k2SW05MlpYSnNZWGtpTENKc2FXNWxYMkZzY0doaElqcDdJblpoYkhWbElqb3hMakI5TENKc2FXNWxYMk52Ykc5eUlqcDdJblpoYkhWbElqb2lZbXhoWTJzaWZTd2liR2x1WlY5a1lYTm9JanBiTkN3MFhTd2liR2x1WlY5M2FXUjBhQ0k2ZXlKMllXeDFaU0k2TW4wc0luQnNiM1FpT201MWJHd3NJbkpsYm1SbGNsOXRiMlJsSWpvaVkzTnpJaXdpY21sbmFIUmZkVzVwZEhNaU9pSnpZM0psWlc0aUxDSjBiM0JmZFc1cGRITWlPaUp6WTNKbFpXNGlmU3dpYVdRaU9pSTBOVFUwTm1ZeE55MW1ZVFl3TFRRMk9XTXRPRFl5TVMxak1tSXlOelV5TXpGak16TWlMQ0owZVhCbElqb2lRbTk0UVc1dWIzUmhkR2x2YmlKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKd2JHOTBJanA3SW1sa0lqb2lNV0ZqTjJZMlpURXROR1kzTWkwME5qSmlMV0prTTJJdE9USXpaR0k1WlRFME1UTTVJaXdpYzNWaWRIbHdaU0k2SWtacFozVnlaU0lzSW5SNWNHVWlPaUpRYkc5MEluMTlMQ0pwWkNJNklqUTNOREV4TjJObUxURXdZekF0TkRKbE9TMDVNRGt4TFRKaFpXWmhPVE5rTTJaak5DSXNJblI1Y0dVaU9pSlFZVzVVYjI5c0luMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltWnZjbTFoZEhSbGNpSTZleUpwWkNJNklqUmhZelF6TldNNExUWmpZVE10TkRoak5pMWlNREU1TFRBM09UQmtNRGcyT0dNd1lpSXNJblI1Y0dVaU9pSkVZWFJsZEdsdFpWUnBZMnRHYjNKdFlYUjBaWElpZlN3aWNHeHZkQ0k2ZXlKcFpDSTZJakZoWXpkbU5tVXhMVFJtTnpJdE5EWXlZaTFpWkROaUxUa3lNMlJpT1dVeE5ERXpPU0lzSW5OMVluUjVjR1VpT2lKR2FXZDFjbVVpTENKMGVYQmxJam9pVUd4dmRDSjlMQ0owYVdOclpYSWlPbnNpYVdRaU9pSmhOVEEyTURNek15MDRPVFV6TFRRd04ySXRPV0k0T0MweU9EQTNZVEZqT0Rka01UWWlMQ0owZVhCbElqb2lSR0YwWlhScGJXVlVhV05yWlhJaWZYMHNJbWxrSWpvaVpUbGpZVFEzTVRVdFlUYzRNUzAwWVRjMkxUaGhOelV0WVdGaU1tVXhZalUzTm1Kaklpd2lkSGx3WlNJNklrUmhkR1YwYVcxbFFYaHBjeUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUp3Ykc5MElqcHVkV3hzTENKMFpYaDBJam9pTkRRd01UTWlmU3dpYVdRaU9pSTFOMkkwTkRSaU5TMDNNRGswTFRReE56WXRZalF5T1MwNVptRXhZMk0wWlRGallqY2lMQ0owZVhCbElqb2lWR2wwYkdVaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW5Cc2IzUWlPbnNpYVdRaU9pSXhZV00zWmpabE1TMDBaamN5TFRRMk1tSXRZbVF6WWkwNU1qTmtZamxsTVRReE16a2lMQ0p6ZFdKMGVYQmxJam9pUm1sbmRYSmxJaXdpZEhsd1pTSTZJbEJzYjNRaWZTd2ljbVZ1WkdWeVpYSnpJanBiZXlKcFpDSTZJalF3TWpka056VTJMVGczTnpBdE5HRXpOQzFoWkdRMUxXSXdOVE5oWmpNd1lqRTJaU0lzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMWRMQ0owYjI5c2RHbHdjeUk2VzFzaVRtRnRaU0lzSWs1RlEwOUdVMTlHVmtOUFRWOVBRMFZCVGw5TlFWTlRRa0ZaWDBaUFVrVkRRVk5VSWwwc1d5SkNhV0Z6SWl3aUxURXVOVGdpWFN4YklsTnJhV3hzSWl3aU1DNHlOeUpkWFgwc0ltbGtJam9pTXpsaE5USmlNelV0WW1ZeU5DMDBNR1l4TFdFeFpqSXRPV1l6TnpaaFlUazNNakF4SWl3aWRIbHdaU0k2SWtodmRtVnlWRzl2YkNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKallXeHNZbUZqYXlJNmJuVnNiSDBzSW1sa0lqb2lZekpsT1dVNFl6QXRaamhoWVMwMFpESTBMV0UzWkRndE5qZGlPVEJoTXpka1pqazVJaXdpZEhsd1pTSTZJa1JoZEdGU1lXNW5aVEZrSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1SaGVYTWlPbHN4TERFMVhYMHNJbWxrSWpvaU1qRTNZamswTjJZdFpEWmhaUzAwT0RVMUxXSTFOMk10TVdOaE1EUTRPV0V5TjJGaUlpd2lkSGx3WlNJNklrUmhlWE5VYVdOclpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liR2x1WlY5allYQWlPaUp5YjNWdVpDSXNJbXhwYm1WZlkyOXNiM0lpT25zaWRtRnNkV1VpT2lJak16STRPR0prSW4wc0lteHBibVZmYW05cGJpSTZJbkp2ZFc1a0lpd2liR2x1WlY5M2FXUjBhQ0k2ZXlKMllXeDFaU0k2Tlgwc0luZ2lPbnNpWm1sbGJHUWlPaUo0SW4wc0lua2lPbnNpWm1sbGJHUWlPaUo1SW4xOUxDSnBaQ0k2SW1aa1lUZGpObU14TFdNek5qTXROREUzTWkwNE5HWXpMV0ppWXpOaFpHRmxNRFUyTVNJc0luUjVjR1VpT2lKTWFXNWxJbjFkTENKeWIyOTBYMmxrY3lJNld5SXhZV00zWmpabE1TMDBaamN5TFRRMk1tSXRZbVF6WWkwNU1qTmtZamxsTVRReE16a2lYWDBzSW5ScGRHeGxJam9pUW05clpXZ2dRWEJ3YkdsallYUnBiMjRpTENKMlpYSnphVzl1SWpvaU1DNHhNaTQxSW4xOU93b2dJQ0FnSUNBZ0lDQWdJQ0FnSUhaaGNpQnlaVzVrWlhKZmFYUmxiWE1nUFNCYmV5SmtiMk5wWkNJNkltVTFNR0UxWlRaa0xUazBPVEl0TkdJd1pDMDVOVFF6TFdaak9XTXhNVE00TWpVNU15SXNJbVZzWlcxbGJuUnBaQ0k2SW1JM1lXTTFabUl4TFRJeFpqa3RORFE0T1MwNU9HVTRMVE5pT0RFM1ptWTFaREkwWmlJc0ltMXZaR1ZzYVdRaU9pSXhZV00zWmpabE1TMDBaamN5TFRRMk1tSXRZbVF6WWkwNU1qTmtZamxsTVRReE16a2lmVjA3Q2lBZ0lDQWdJQ0FnSUNBZ0lDQWdDaUFnSUNBZ0lDQWdJQ0FnSUNBZ1FtOXJaV2d1WlcxaVpXUXVaVzFpWldSZmFYUmxiWE1vWkc5amMxOXFjMjl1TENCeVpXNWtaWEpmYVhSbGJYTXBPd29nSUNBZ0lDQWdJQ0FnSUNCOUtUc0tJQ0FnSUNBZ0lDQWdJSDA3Q2lBZ0lDQWdJQ0FnSUNCcFppQW9aRzlqZFcxbGJuUXVjbVZoWkhsVGRHRjBaU0FoUFNBaWJHOWhaR2x1WnlJcElHWnVLQ2s3Q2lBZ0lDQWdJQ0FnSUNCbGJITmxJR1J2WTNWdFpXNTBMbUZrWkVWMlpXNTBUR2x6ZEdWdVpYSW9Ja1JQVFVOdmJuUmxiblJNYjJGa1pXUWlMQ0JtYmlrN0NpQWdJQ0FnSUNBZ2ZTa29LVHNLSUNBZ0lDQWdJQ0FLSUNBZ0lDQWdJQ0E4TDNOamNtbHdkRDRLSUNBZ0lEd3ZZbTlrZVQ0S1BDOW9kRzFzUGc9PSIgd2lkdGg9Ijc5MCIgc3R5bGU9ImJvcmRlcjpub25lICFpbXBvcnRhbnQ7IiBoZWlnaHQ9IjMzMCI+PC9pZnJhbWU+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF81N2E4MjQyZGUyYTI0ZjMwOGYxMGRkOTM4MzhkODEwNS5zZXRDb250ZW50KGlfZnJhbWVfNDg4ZjE1YjYxOTQ5NDIyY2E4Y2EzMDU4NTY1NGE4MDIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl81Nzg4NGJhM2FkOTk0YmJiYjk2M2U0NzczODQyNjQzMS5iaW5kUG9wdXAocG9wdXBfNTdhODI0MmRlMmEyNGYzMDhmMTBkZDkzODM4ZDgxMDUpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZTNkYTNjZDkyZmM0NDBmMzhhYWZmYTEyNzQ5M2ZhOGQgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zNTQ4LC03MS4wNTM0XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfNjkzOWUxN2MyODY5NDI5ZGIwMzIzNWFlMDE0Y2VlOGIpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzk1NDQ4NmU1NzU5YTRkNTRhNTY2NmYxMzk2OTQxZjFkID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnc3RhdHMnLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2UzZGEzY2Q5MmZjNDQwZjM4YWFmZmExMjc0OTNmYThkLnNldEljb24oaWNvbl85NTQ0ODZlNTc1OWE0ZDU0YTU2NjZmMTM5Njk0MWYxZCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF84Y2U4NTM0ZDU0MDI0YTBlYmUzMGY2M2M5MWFiZDJkNyA9IEwucG9wdXAoe21heFdpZHRoOiAnMjY1MCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGlfZnJhbWVfMjEwYWIyNDVhNmQxNGYwOTk3MWFiMmVjMTliZjQ0NzQgPSAkKCc8aWZyYW1lIHNyYz0iZGF0YTp0ZXh0L2h0bWw7Y2hhcnNldD11dGYtODtiYXNlNjQsQ2lBZ0lDQUtQQ0ZFVDBOVVdWQkZJR2gwYld3K0NqeG9kRzFzSUd4aGJtYzlJbVZ1SWo0S0lDQWdJRHhvWldGa1Bnb2dJQ0FnSUNBZ0lEeHRaWFJoSUdOb1lYSnpaWFE5SW5WMFppMDRJajRLSUNBZ0lDQWdJQ0E4ZEdsMGJHVStPRFEwTXprM01Ed3ZkR2wwYkdVK0NpQWdJQ0FnSUNBZ0NqeHNhVzVySUhKbGJEMGljM1I1YkdWemFHVmxkQ0lnYUhKbFpqMGlhSFIwY0hNNkx5OWpaRzR1Y0hsa1lYUmhMbTl5Wnk5aWIydGxhQzl5Wld4bFlYTmxMMkp2YTJWb0xUQXVNVEl1TlM1dGFXNHVZM056SWlCMGVYQmxQU0owWlhoMEwyTnpjeUlnTHo0S0lDQWdJQ0FnSUNBS1BITmpjbWx3ZENCMGVYQmxQU0owWlhoMEwycGhkbUZ6WTNKcGNIUWlJSE55WXowaWFIUjBjSE02THk5alpHNHVjSGxrWVhSaExtOXlaeTlpYjJ0bGFDOXlaV3hsWVhObEwySnZhMlZvTFRBdU1USXVOUzV0YVc0dWFuTWlQand2YzJOeWFYQjBQZ284YzJOeWFYQjBJSFI1Y0dVOUluUmxlSFF2YW1GMllYTmpjbWx3ZENJK0NpQWdJQ0JDYjJ0bGFDNXpaWFJmYkc5blgyeGxkbVZzS0NKcGJtWnZJaWs3Q2p3dmMyTnlhWEIwUGdvZ0lDQWdJQ0FnSUR4emRIbHNaVDRLSUNBZ0lDQWdJQ0FnSUdoMGJXd2dld29nSUNBZ0lDQWdJQ0FnSUNCM2FXUjBhRG9nTVRBd0pUc0tJQ0FnSUNBZ0lDQWdJQ0FnYUdWcFoyaDBPaUF4TURBbE93b2dJQ0FnSUNBZ0lDQWdmUW9nSUNBZ0lDQWdJQ0FnWW05a2VTQjdDaUFnSUNBZ0lDQWdJQ0FnSUhkcFpIUm9PaUE1TUNVN0NpQWdJQ0FnSUNBZ0lDQWdJR2hsYVdkb2REb2dNVEF3SlRzS0lDQWdJQ0FnSUNBZ0lDQWdiV0Z5WjJsdU9pQmhkWFJ2T3dvZ0lDQWdJQ0FnSUNBZ2ZRb2dJQ0FnSUNBZ0lEd3ZjM1I1YkdVK0NpQWdJQ0E4TDJobFlXUStDaUFnSUNBOFltOWtlVDRLSUNBZ0lDQWdJQ0FLSUNBZ0lDQWdJQ0E4WkdsMklHTnNZWE56UFNKaWF5MXliMjkwSWo0S0lDQWdJQ0FnSUNBZ0lDQWdQR1JwZGlCamJHRnpjejBpWW1zdGNHeHZkR1JwZGlJZ2FXUTlJall4TjJFNE1XRm1MV00xT0RRdE5EVXdPUzA0WkRjMkxUSTJZbUkxTURaaE4yWmpaU0krUEM5a2FYWStDaUFnSUNBZ0lDQWdQQzlrYVhZK0NpQWdJQ0FnSUNBZ0NpQWdJQ0FnSUNBZ1BITmpjbWx3ZENCMGVYQmxQU0owWlhoMEwycGhkbUZ6WTNKcGNIUWlQZ29nSUNBZ0lDQWdJQ0FnSUNBb1puVnVZM1JwYjI0b0tTQjdDaUFnSUNBZ0lDQWdJQ0IyWVhJZ1ptNGdQU0JtZFc1amRHbHZiaWdwSUhzS0lDQWdJQ0FnSUNBZ0lDQWdRbTlyWldndWMyRm1aV3g1S0daMWJtTjBhVzl1S0NrZ2V3b2dJQ0FnSUNBZ0lDQWdJQ0FnSUhaaGNpQmtiMk56WDJwemIyNGdQU0I3SWpRM016WTJZelJoTFRFeFltSXRORGMxTUMwNE1XWTFMVEZtTkRNMk5UUm1NVE5qT0NJNmV5SnliMjkwY3lJNmV5SnlaV1psY21WdVkyVnpJanBiZXlKaGRIUnlhV0oxZEdWeklqcDdJbTF2Ym5Sb2N5STZXekFzTmwxOUxDSnBaQ0k2SWpCak56QXpPR1l4TFRoaFpqQXRORGhqTkMxaU5EazBMVGs1TW1VeFpERTJObVU1WWlJc0luUjVjR1VpT2lKTmIyNTBhSE5VYVdOclpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liR2x1WlY5allYQWlPaUp5YjNWdVpDSXNJbXhwYm1WZlkyOXNiM0lpT25zaWRtRnNkV1VpT2lJalpUWm1OVGs0SW4wc0lteHBibVZmYW05cGJpSTZJbkp2ZFc1a0lpd2liR2x1WlY5M2FXUjBhQ0k2ZXlKMllXeDFaU0k2Tlgwc0luZ2lPbnNpWm1sbGJHUWlPaUo0SW4wc0lua2lPbnNpWm1sbGJHUWlPaUo1SW4xOUxDSnBaQ0k2SW1JMlkyRTRPV1U0TFRaaFpqSXRORGN4T1MxaE56Tm1MV1E0TVRsaE4yVXpPR1k0WXlJc0luUjVjR1VpT2lKTWFXNWxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhwYm1WZllXeHdhR0VpT25zaWRtRnNkV1VpT2pBdU1YMHNJbXhwYm1WZlkyRndJam9pY205MWJtUWlMQ0pzYVc1bFgyTnZiRzl5SWpwN0luWmhiSFZsSWpvaUl6Rm1OemRpTkNKOUxDSnNhVzVsWDJwdmFXNGlPaUp5YjNWdVpDSXNJbXhwYm1WZmQybGtkR2dpT25zaWRtRnNkV1VpT2pWOUxDSjRJanA3SW1acFpXeGtJam9pZUNKOUxDSjVJanA3SW1acFpXeGtJam9pZVNKOWZTd2lhV1FpT2lJM09UWTRabVJpT0MxallUYzBMVFEzT1RndE9UbGhOeTAzTTJaaVpXTmhOamt4WldRaUxDSjBlWEJsSWpvaVRHbHVaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZlMzBzSW1sa0lqb2lNVGN4T1RkbU9HUXRNVGN6WVMwME56aG1MVGc1TURNdFpEaGhaalU0T0RBd1lqY3hJaXdpZEhsd1pTSTZJa1JoZEdWMGFXMWxWR2xqYTBadmNtMWhkSFJsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKdWRXMWZiV2x1YjNKZmRHbGphM01pT2pWOUxDSnBaQ0k2SWpjeFlqRTRPR0kyTFRjelptRXROREprWXkwNU9ESmxMVGRsTmpJME5URXlObUkyTmlJc0luUjVjR1VpT2lKRVlYUmxkR2x0WlZScFkydGxjaUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUprWVhSaFgzTnZkWEpqWlNJNmV5SnBaQ0k2SWpabE1UWTBZalkxTFRobE0ySXRORGswT1MwNE5ERXdMVGs1TjJZMk56TmhNMlUzTVNJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc0ltZHNlWEJvSWpwN0ltbGtJam9pTkdSbU1qQXhZVE10Wm1Wak15MDBOMkkzTFdKaFlqWXRZVFkxTWpZeFpUSTNOMlV3SWl3aWRIbHdaU0k2SWt4cGJtVWlmU3dpYUc5MlpYSmZaMng1Y0dnaU9tNTFiR3dzSW0xMWRHVmtYMmRzZVhCb0lqcHVkV3hzTENKdWIyNXpaV3hsWTNScGIyNWZaMng1Y0dnaU9uc2lhV1FpT2lKa1lqSXpZbU5qTkMxak9HWTFMVFF4TXpVdFltTXpaUzAxT0dVMlpERXpNelJtTlRnaUxDSjBlWEJsSWpvaVRHbHVaU0o5TENKelpXeGxZM1JwYjI1ZloyeDVjR2dpT201MWJHeDlMQ0pwWkNJNklqTmlaREl4TW1FekxURTBNVGd0TkRjeVpDMDRZV1kyTFRsa1ptVXhOekZtTm1Fd055SXNJblI1Y0dVaU9pSkhiSGx3YUZKbGJtUmxjbVZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW5Cc2IzUWlPbnNpYVdRaU9pSm1PVEZoWkdabVpTMHdZakpqTFRRMk9HSXRZVFk0T0MwMk16aGtPV0ZpWkRBM056SWlMQ0p6ZFdKMGVYQmxJam9pUm1sbmRYSmxJaXdpZEhsd1pTSTZJbEJzYjNRaWZYMHNJbWxrSWpvaVltSmxPRGhsWXpJdFlXUmtaQzAwTm1VMUxUazFNbUl0TVRnNE5EUXlOMlU1T0RVNElpd2lkSGx3WlNJNklsSmxjMlYwVkc5dmJDSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNhVzVsWDJGc2NHaGhJanA3SW5aaGJIVmxJam93TGpGOUxDSnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2ZXlKMllXeDFaU0k2SWlNeFpqYzNZalFpZlN3aWJHbHVaVjlxYjJsdUlqb2ljbTkxYm1RaUxDSnNhVzVsWDNkcFpIUm9JanA3SW5aaGJIVmxJam8xZlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pTkRZeE5XVTBZakl0TVRVNFl5MDBZVGRqTFdJMFlUUXRPV000TXpVM1lqUmlPRFl5SWl3aWRIbHdaU0k2SWt4cGJtVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2lZMkZzYkdKaFkyc2lPbTUxYkd4OUxDSnBaQ0k2SW1Oak5EVTBNV0UxTFdOaU5HVXRORGRqWWkxaU5EVXlMVEZpTjJFNE5ETTRZakl5WmlJc0luUjVjR1VpT2lKRVlYUmhVbUZ1WjJVeFpDSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnRiMjUwYUhNaU9sc3dMRFFzT0YxOUxDSnBaQ0k2SW1aa05UVTRNalJrTFdRelpUQXRORGM0WWkwNE1tSmpMVGRqT1RkbU9XRTRPRFl5WkNJc0luUjVjR1VpT2lKTmIyNTBhSE5VYVdOclpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2lZV04wYVhabFgyUnlZV2NpT2lKaGRYUnZJaXdpWVdOMGFYWmxYM05qY205c2JDSTZJbUYxZEc4aUxDSmhZM1JwZG1WZmRHRndJam9pWVhWMGJ5SXNJblJ2YjJ4eklqcGJleUpwWkNJNkltWTNaRGRpWTJKaExUaGtNakl0TkdZd01DMDVObVl3TFRNMFpEazJOVEV4TjJNNE9TSXNJblI1Y0dVaU9pSlFZVzVVYjI5c0luMHNleUpwWkNJNkltRm1OMlZtT1RVeUxUUTNaRE10TkRJeVl5MWhNek0zTFRoak9Ua3hZbU0yWmpWbE9DSXNJblI1Y0dVaU9pSkNiM2hhYjI5dFZHOXZiQ0o5TEhzaWFXUWlPaUppWW1VNE9HVmpNaTFoWkdSa0xUUTJaVFV0T1RVeVlpMHhPRGcwTkRJM1pUazROVGdpTENKMGVYQmxJam9pVW1WelpYUlViMjlzSW4wc2V5SnBaQ0k2SW1VNVlXTmtaamMzTFRrd1pqQXRORFppTnkxaVpXSTVMVGhpWlRjd09ERmxOV05sTXlJc0luUjVjR1VpT2lKSWIzWmxjbFJ2YjJ3aWZTeDdJbWxrSWpvaVpEWTJNREl3WW1ZdE16WmlPUzAwTVRFNExUbG1ZVFF0TkRGa1pqYzJZekl4TmpOa0lpd2lkSGx3WlNJNklraHZkbVZ5Vkc5dmJDSjlMSHNpYVdRaU9pSXpaR0ZrTVdZMlppMHpNbU01TFRRM05ESXRPV013TkMwM1lqazVORFZtWXpCak5tTWlMQ0owZVhCbElqb2lTRzkyWlhKVWIyOXNJbjBzZXlKcFpDSTZJamxqTVRGbE56Rm1MV015Wm1RdE5ERmlOUzFpTWpGaExUa3lNREZrTWpFNVpXWmhNQ0lzSW5SNWNHVWlPaUpJYjNabGNsUnZiMndpZlYxOUxDSnBaQ0k2SWpNeU1UWmxaamN4TFRrM01ESXRORFZoT1MwNU1HSTFMVEV5WmpoaFlUY3dOamMwT1NJc0luUjVjR1VpT2lKVWIyOXNZbUZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW5Cc2IzUWlPbnNpYVdRaU9pSm1PVEZoWkdabVpTMHdZakpqTFRRMk9HSXRZVFk0T0MwMk16aGtPV0ZpWkRBM056SWlMQ0p6ZFdKMGVYQmxJam9pUm1sbmRYSmxJaXdpZEhsd1pTSTZJbEJzYjNRaWZYMHNJbWxrSWpvaVpqZGtOMkpqWW1FdE9HUXlNaTAwWmpBd0xUazJaakF0TXpSa09UWTFNVEUzWXpnNUlpd2lkSGx3WlNJNklsQmhibFJ2YjJ3aWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW5Cc2IzUWlPbnNpYVdRaU9pSm1PVEZoWkdabVpTMHdZakpqTFRRMk9HSXRZVFk0T0MwMk16aGtPV0ZpWkRBM056SWlMQ0p6ZFdKMGVYQmxJam9pUm1sbmRYSmxJaXdpZEhsd1pTSTZJbEJzYjNRaWZTd2ljbVZ1WkdWeVpYSnpJanBiZXlKcFpDSTZJak5pWkRJeE1tRXpMVEUwTVRndE5EY3laQzA0WVdZMkxUbGtabVV4TnpGbU5tRXdOeUlzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMWRMQ0owYjI5c2RHbHdjeUk2VzFzaVRtRnRaU0lzSWtjeFgxTlRWRjlIVEU5Q1FVd2lYU3hiSWtKcFlYTWlMQ0l0TUM0ME1DSmRMRnNpVTJ0cGJHd2lMQ0l3TGpNNElsMWRmU3dpYVdRaU9pSmxPV0ZqWkdZM055MDVNR1l3TFRRMllqY3RZbVZpT1MwNFltVTNNRGd4WlRWalpUTWlMQ0owZVhCbElqb2lTRzkyWlhKVWIyOXNJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbU5oYkd4aVlXTnJJanB1ZFd4c0xDSndiRzkwSWpwN0ltbGtJam9pWmpreFlXUm1abVV0TUdJeVl5MDBOamhpTFdFMk9EZ3ROak00WkRsaFltUXdOemN5SWl3aWMzVmlkSGx3WlNJNklrWnBaM1Z5WlNJc0luUjVjR1VpT2lKUWJHOTBJbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lKaU5qY3lZV1UyWXkxaE56SXdMVFEzTWpVdE9XSTBNQzFrT1dNNU5UaGpORE00TlRnaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFN3aWRHOXZiSFJwY0hNaU9sdGJJazVoYldVaUxDSlBRbE5mUkVGVVFTSmRMRnNpUW1saGN5SXNJazVCSWwwc1d5SlRhMmxzYkNJc0lrNUJJbDFkZlN3aWFXUWlPaUk1WXpFeFpUY3haaTFqTW1aa0xUUXhZalV0WWpJeFlTMDVNakF4WkRJeE9XVm1ZVEFpTENKMGVYQmxJam9pU0c5MlpYSlViMjlzSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1admNtMWhkSFJsY2lJNmV5SnBaQ0k2SWpFM01UazNaamhrTFRFM00yRXRORGM0WmkwNE9UQXpMV1E0WVdZMU9EZ3dNR0kzTVNJc0luUjVjR1VpT2lKRVlYUmxkR2x0WlZScFkydEdiM0p0WVhSMFpYSWlmU3dpY0d4dmRDSTZleUpwWkNJNkltWTVNV0ZrWm1abExUQmlNbU10TkRZNFlpMWhOamc0TFRZek9HUTVZV0prTURjM01pSXNJbk4xWW5SNWNHVWlPaUpHYVdkMWNtVWlMQ0owZVhCbElqb2lVR3h2ZENKOUxDSjBhV05yWlhJaU9uc2lhV1FpT2lJM01XSXhPRGhpTmkwM00yWmhMVFF5WkdNdE9UZ3laUzAzWlRZeU5EVXhNalppTmpZaUxDSjBlWEJsSWpvaVJHRjBaWFJwYldWVWFXTnJaWElpZlgwc0ltbGtJam9pTjJJM1pqazVORFF0TVRRd1pDMDBaak14TFdFd1pURXRaRGM0Tm1KalpEYzBNV0ZtSWl3aWRIbHdaU0k2SWtSaGRHVjBhVzFsUVhocGN5SjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2ZXlKMllXeDFaU0k2SWlNNU9XUTFPVFFpZlN3aWJHbHVaVjlxYjJsdUlqb2ljbTkxYm1RaUxDSnNhVzVsWDNkcFpIUm9JanA3SW5aaGJIVmxJam8xZlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pT0dZMlpqSTNPVGt0WVRsaVl5MDBaV0kyTFRrMVpHTXRNV1F5WVRjNU1tWTJOakUzSWl3aWRIbHdaU0k2SWt4cGJtVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liR2x1WlY5allYQWlPaUp5YjNWdVpDSXNJbXhwYm1WZlkyOXNiM0lpT25zaWRtRnNkV1VpT2lJak16STRPR0prSW4wc0lteHBibVZmYW05cGJpSTZJbkp2ZFc1a0lpd2liR2x1WlY5M2FXUjBhQ0k2ZXlKMllXeDFaU0k2Tlgwc0luZ2lPbnNpWm1sbGJHUWlPaUo0SW4wc0lua2lPbnNpWm1sbGJHUWlPaUo1SW4xOUxDSnBaQ0k2SWpSa1pqSXdNV0V6TFdabFl6TXRORGRpTnkxaVlXSTJMV0UyTlRJMk1XVXlOemRsTUNJc0luUjVjR1VpT2lKTWFXNWxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbVJoZEdGZmMyOTFjbU5sSWpwN0ltbGtJam9pTnpOallqTTNNVGt0TldGaVlpMDBaak5tTFRreVlUZ3ROak5oWWpKa1lURTVPV1EzSWl3aWRIbHdaU0k2SWtOdmJIVnRia1JoZEdGVGIzVnlZMlVpZlN3aVoyeDVjR2dpT25zaWFXUWlPaUppTm1OaE9EbGxPQzAyWVdZeUxUUTNNVGt0WVRjelppMWtPREU1WVRkbE16aG1PR01pTENKMGVYQmxJam9pVEdsdVpTSjlMQ0pvYjNabGNsOW5iSGx3YUNJNmJuVnNiQ3dpYlhWMFpXUmZaMng1Y0dnaU9tNTFiR3dzSW01dmJuTmxiR1ZqZEdsdmJsOW5iSGx3YUNJNmV5SnBaQ0k2SWpjNU5qaG1aR0k0TFdOaE56UXRORGM1T0MwNU9XRTNMVGN6Wm1KbFkyRTJPVEZsWkNJc0luUjVjR1VpT2lKTWFXNWxJbjBzSW5ObGJHVmpkR2x2Ymw5bmJIbHdhQ0k2Ym5Wc2JIMHNJbWxrSWpvaU5HRmxNemhoTUdNdFpEazNZeTAwWkRRMExUaGlNakV0TkRNNE5tTXpZelV3WldVMElpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWTJGc2JHSmhZMnNpT201MWJHeDlMQ0pwWkNJNkltSmpaVGc1WTJaaExXSmpaVGd0TkRObVpTMWhOalJpTFRFMllqVmhNVGRqWTJGak5DSXNJblI1Y0dVaU9pSkVZWFJoVW1GdVoyVXhaQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpqWVd4c1ltRmpheUk2Ym5Wc2JDd2lZMjlzZFcxdVgyNWhiV1Z6SWpwYkluZ2lMQ0o1SWwwc0ltUmhkR0VpT25zaWVDSTZleUpmWDI1a1lYSnlZWGxmWHlJNklrRkJSRUZXTlU4elpGVkpRVUZMYWtkc2NtUXhVV2RCUVd0RVYyRjBNMVpEUVVGQ05IQktNak5rVlVsQlFVZEJWRzlpWkRGUlowRkJVMGxMYTNRelZrTkJRVUYzT0dGbE0yUlZTVUZCUW1obmNUZGtNVkZuUVVGQlRTdDFkRE5XUTBGQlJHOVFZa3N6WkZWSlFVRk9RM04wWW1ReFVXZEJRWFZDZFRWME0xWkRRVUZEWjJseWVUTmtWVWxCUVVscU5YWTNaREZSWjBGQlkwZHFSSFF6VmtOQlFVSlpNVGhoTTJSVlNVRkJSVUpIZVhKa01WRm5RVUZMVEZoT2RETldRMEZCUVZGS1RrY3paRlZKUVVGUWFWTXhUR1F4VVdkQlFUUkJTRmwwTTFaRFFVRkVTV05PZFROa1ZVbEJRVXhFWmpOeVpERlJaMEZCYlVVM2FYUXpWa05CUVVOQmRtVlhNMlJWU1VGQlIyZHpObUprTVZGblFVRlZTblp6ZEROV1EwRkJRVFJEZGtNelpGVkpRVUZEUWpVNE4yUXhVV2RCUVVOUGFqSjBNMVpEUVVGRWQxWjJjVE5rVlVsQlFVNXFSaTlpWkRGUlowRkJkMFJSUW5WSVZrTkJRVU52YjNkVE5HUlZTVUZCU2tGVFEweG9NVkZuUVVGbFNVVk1kVWhXUTBGQlFtYzRRVFkwWkZWSlFVRkZhR1pGY21neFVXZEJRVTFOTkZaMVNGWkRRVUZCV1ZCU2JUUmtWVWxCUVVGRGMwaE1hREZSWjBGQk5rSnZaM1ZJVmtOQlFVUlJhVk5QTkdSVlNVRkJUR28wU25Kb01WRm5RVUZ2UjJOeGRVaFdRMEZCUTBreGFUSTBaRlZKUVVGSVFrWk5ZbWd4VVdkQlFWZE1VVEIxU0ZaRFFVRkNRVWw2YVRSa1ZVbEJRVU5wVTA4M2FERlJaMEZCUlVGRkwzVklWa05CUVVRMFlqQkxOR1JWU1VGQlQwUmxVbUpvTVZGblFVRjVSVEZLZFVoV1EwRkJRM2QyUlhrMFpGVkpRVUZLWjNKVlRHZ3hVV2RCUVdkS2NGUjFTRlpEUVVGQ2IwTldaVFJrVlVsQlFVWkNORmR5YURGUlowRkJUMDlrWkhWSVZrTkJRVUZuVm0xSE5HUlZTVUZCUVdwR1dreG9NVkZuUVVFNFJFNXZkVWhXUTBGQlJGbHZiWFUwWkZWSlFVRk5RVkppTjJneFVXZEJRWEZKUW5sMVNGWkRRVUZEVVRjelZ6UmtWVWxCUVVob1pXVmlhREZSWjBGQldVMHhPSFZJVmtOQlFVSkpVRWxETkdSVlNVRkJSRU55Wnpkb01WRm5RVUZIUW5GSWRVaFdRMEZCUVVGcFdYRTBaRlZKUVVGUGFqTnFZbWd4VVdkQlFUQkhZVkoxU0ZaRFFVRkROREZhVXpSa1ZVbEJRVXRDUlcxTWFERlJaMEZCYVV4UFluVklWa05CUVVKM1NYQXJOR1JWU1VGQlJtbFNiM0pvTVZGblFVRlJRVU50ZFVoV1EwRkJRVzlpTm0wMFpGVkpRVUZDUkdWeVRHZ3hVV2RCUVN0RmVYZDFTRlpEUVVGRVozVTNUelJrVlVsQlFVMW5jWFEzYURGUlowRkJjMHB0Tm5WSVZrTkJRVU5aUTB3Mk5HUlZTVUZCU1VJemQySm9NVkZuUVVGaFQySkZkVWhXUTBGQlFsRldZMmswWkZWSlFVRkVha1Y1TjJneFVXZEJRVWxFVUZCMVNGWkRRVUZCU1c5MFN6UmtWVWxCUVZCQlVURnlhREZSWjBGQk1rZ3ZXblZJVmtOQlFVUkJOM1I1TkdSVlNVRkJTMmhrTkV4b01WRm5RVUZyVFhwcWRVaFdRMEZCUWpSUEsyVTBaRlZKUVVGSFEzRTJjbWd4VVdkQlFWTkNiblYxU0ZaRFFVRkJkMmxRUnpSa1ZVbEJRVUpxTXpsTWFERlJaMEZCUVVkaU5IVklWa05CUVVSdk1WQjFOR1JWU1VGQlRrSkVMemRvTVZGblFVRjFURWxEZFZoV1EwRkJRMmRKVVdFMVpGVkpRVUZKYVZGRFltd3hVV2RCUVdOUU9FMTFXRlpEUVVGQ1dXSm9RelZrVlVsQlFVVkVaRVUzYkRGUlowRkJTMFYzV0hWWVZrTkJRVUZSZFhoeE5XUlZTVUZCVUdkd1NISnNNVkZuUVVFMFNtZG9kVmhXUTBGQlJFbENlVmMxWkZWSlFVRk1RakpMVEd3eFVXZEJRVzFQVlhKMVdGWkRRVUZEUVZaREt6VmtWVWxCUVVkcVJFMXliREZSWjBGQlZVUkpNblZZVmtOQlFVRTBiMVJ0TldSVlNVRkJRMEZSVUdKc01WRm5RVUZEU0RsQmRWaFdRMEZCUkhjM1ZVODFaRlZKUVVGT2FHTlNOMnd4VVdkQlFYZE5kRXQxV0ZaRFFVRkRiMDlyTmpWa1ZVbEJRVXBEY0ZWaWJERlJaMEZCWlVKb1ZuVllWa05CUVVKbmFERnBOV1JWU1VGQlJXb3lWemRzTVZGblFVRk5SMVptZFZoV1EwRkJRVmt4UjBzMVpGVkpRVUZCUWtSYWNtd3hVV2RCUVRaTVJuQjFXRlpEUVVGRVVVbEhNalZrVlVsQlFVeHBVR05NYkRGUlowRkJiMUExZW5WWVZrTkJRVU5KWWxobE5XUlZTVUZCU0VSalpYSnNNVkZuUVVGWFJYUXJkVmhXUXlJc0ltUjBlWEJsSWpvaVpteHZZWFEyTkNJc0luTm9ZWEJsSWpwYk1UUTBYWDBzSW5raU9uc2lYMTl1WkdGeWNtRjVYMThpT2lKQlFVRkJTVUp0VDBsVlFVRkJRVU5CWmpWQmFGRkJRVUZCUVVSdGEybEdRVUZCUVVGWlJYbFdTVlZCUVVGQlFVRklOV05vVVVGQlFVRkpSSGh0UTBaQlFVRkJRVWxOVTJGSlZVRkJRVUZCWjFGd1ZXaFJRVUZCUVVORVFXcDVSa0ZCUVVGQlNVUTJTMGxWUVVGQlFVSm5WMWxOYUZGQlFVRkJTVUl3WmtOR1FVRkJRVUYzU1RreFNWVkJRVUZCUkdkQlNVVm9VVUZCUVVGQlFubHFRMFpCUVVGQlFVbFBUMWhKVlVGQlFVRkNRVm8zVldoUlFVRkJRVVZFY2pCcFJrRkJRVUZCV1VjdmQwbFZRVUZCUVVSQk55dHZhRkZCUVVGQlJVSjNOVk5HUVVGQlFVRnZVRVJtU1ZWQlFVRkJSRUZzT1RCb1VVRkJRVUZOUVNzeWVVWkJRVUZCUVRSUFdGbEpWVUZCUVVGRVFVYzNiMmhSUVVGQlFVbENVbTE1UmtGQlFVRkJXVWxrT0VsVlFVRkJRVVJCWlhwcmFGRkJRVUZCUVVKM09XbENRVUZCUVVGWlIxTjZTVVZCUVVGQlFtZG5jamhuVVVGQlFVRkpRMmQ1ZVVKQlFVRkJRV2RNTjFoSlJVRkJRVUZCUVhKMGIyZFJRVUZCUVVsRFpETlRRa0ZCUVVGQlFVa3paMGxGUVVGQlFVRm5aMlZqWjFGQlFVRkJSMEl4TjJsQ1FVRkJRVUZuUjI0eFNVVkJRVUZCUkVGelpFVm5VVUZCUVVGQlJEWnlVMEpCUVVGQlFWRkZTMHRKUlVGQlFVRkNRVVkxU1dkUlFVRkJRVU5FYzIxVFFrRkJRVUZCU1UxSGFFbEZRVUZCUVVSbk5FcG5aMUZCUVVGQlRVRkJhME5DUVVGQlFVRm5RME5JU1VWQlFVRkJSR2RWYjFGblVVRkJRVUZGUTBablUwSkJRVUZCUVc5TVpDdEpSVUZCUVVGQ1FVOUZXV2RSUVVGQlFVRkROVVJUUWtGQlFVRkJVVWhQY1Vnd1FVRkJRVUpuVW1KSlpsRkJRVUZCU1VGWWRXZzVRVUZCUVVGdlQyNUNTREJCUVVGQlFXY3dZemhtVVVGQlFVRkpRelF6VWpsQlFVRkJRVUZMUkhKSU1FRkJRVUZEUVhCMldXWlJRVUZCUVVsRVYwRkRRa0ZCUVVGQmQwWnJSMGxGUVVGQlFVUkJVV3BOWjFGQlFVRkJTMEZ5V1VOQ1FVRkJRVUZ2UWxOT1NVVkJRVUZCUWtGcFdsVm5VVUZCUVVGTlJEbHVVMEpCUVVGQlFWbElTMjFKUlVGQlFVRkJaMVkxUldkUlFVRkJRVTlCTjJaRFFrRkJRVUZCYjBOQ2JrbEZRVUZCUVVGQk9XMU5aMUZCUVVGQlJVUk1XVU5DUVVGQlFVRnZTMEprU1VWQlFVRkJRVUZUTVdkblVVRkJRVUZGUkRGVmFVSkJRVUZCUVc5S09VNUpSVUZCUVVGQ1oxaHNSV2RSUVVGQlFVRkJaRlpUUWtGQlFVRkJkMDUwV1VsRlFVRkJRVUpuWWpGVloxRkJRVUZCUTBGRVZXbENRVUZCUVVGM1NscFBTVVZCUVVGQlFXZDJWa2xuVVVGQlFVRkpSR3BXYVVKQlFVRkJRVFJCYkdKSlJVRkJRVUZEUVV0d1VXZFJRVUZCUVVGQ1RIcFRRa0ZCUVVGQmIwZHpSMGxWUVVGQlFVUm5Vbms0YUZGQlFVRkJSVUZyVjBOR1FVRkJRVUZuUVVOQ1NWVkJRVUZCUkVGT2JrbG9VVUZCUVVGUFFuTlplVVpCUVVGQlFVbExUbFZKVlVGQlFVRkJaMjlWWjJoUlFVRkJRVUZEWmxCRFJrRkJRVUZCUVVvd2QwbFZRVUZCUVVGbmFta3dhRkZCUVVGQlIwSXZTMmxHUVVGQlFVRm5TRUZ1U1ZWQlFVRkJRbWR1ZUZsb1VVRkJRVUZEUkU5Q1UwWkJRVUZCUVVGUU16QkpSVUZCUVVGRFoybG1iMmRSUVVGQlFVTkJWMEZEUmtGQlFVRkJkMHRKUmtsVlFVRkJRVUZuWmxGcmFGRkJRVUZCUjBKWVJGTkdRVUZCUVVGM1JFVlNTVlZCUVVGQlFVRTVlRzlvVVVGQlFVRkZRemhLUTBaQlFVRkJRV2RKUlhWSlZVRkJRVUZEUVVOVE9HaFJRVUZCUVVkRFVreDVSa0ZCUVVGQldVSnJkMGxWUVVGQlFVUm5WMU4zYUZGQlFVRkJSVU5oUzBOR1FVRkJRVUYzVG05clNWVkJRVUZCUTJkbVFuZG9VVUZCUVVGSlFXVkdRMFpCUVVGQlFWbE5RVXhKVlVGQlFVRkVaMFJuTUdoUlFVRkJRVWRDWkVScFJrRkJRVUZCTkV0elVFbFZRVUZCUVVSbk5tZDNhRkZCUVVGQlRVRndRMmxHUVVGQlFVRjNSMmRJU1ZWQlFVRkJRMEZhWjFGb1VVRkJRVUZEUW10QlUwWkJRVUZCUVRSSFNDdEpSVUZCUVVGRVFXMVFOR2RSUVVGQlFVdEVVQzlwUWtGQlFVRkJaMEZpTDBsRlFVRkJRVUpuUkVGcmFGRkJRVUZCUTBGVFJYbEdRVUZCUVVGQlFtZGtTVlZCUVVGQlFrRk1NRkZvVVVGQlFVRkhRa2RoZVVaQlFVRkJRVzlHTWxOSlZVRkJRVUZEWjFoYVNXaFJRVUZCUVV0Q1pHdHBSa0VpTENKa2RIbHdaU0k2SW1ac2IyRjBOalFpTENKemFHRndaU0k2V3pFME5GMTlmWDBzSW1sa0lqb2lOMk0yTVRVeFpERXRabVEyTXkwMFpETXlMVGs0WmpndE1tVmpaV1JpWlRZME1URXpJaXdpZEhsd1pTSTZJa052YkhWdGJrUmhkR0ZUYjNWeVkyVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2lZbUZ6WlNJNk1qUXNJbTFoYm5ScGMzTmhjeUk2V3pFc01pdzBMRFlzT0N3eE1sMHNJbTFoZUY5cGJuUmxjblpoYkNJNk5ETXlNREF3TURBdU1Dd2liV2x1WDJsdWRHVnlkbUZzSWpvek5qQXdNREF3TGpBc0ltNTFiVjl0YVc1dmNsOTBhV05yY3lJNk1IMHNJbWxrSWpvaU1HTm1NR0ZsTkRZdE5qQmlaQzAwWVRrMExUZzVORE10T0RVMU5UWXhPV1prWm1ZNUlpd2lkSGx3WlNJNklrRmtZWEIwYVhabFZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3ZlN3aWFXUWlPaUptTnpVd1lXUmxOeTA1Wm1abExUUTVZMk10T0RJM1pDMWpaRE0xWkRZMVpqWTFPVFVpTENKMGVYQmxJam9pVkc5dmJFVjJaVzUwY3lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZTMwc0ltbGtJam9pWlRSa09HSTVNekl0WVdaa1l5MDBOelpoTFdJNE5HUXRObUpsTW1FM05XTXpOR1kwSWl3aWRIbHdaU0k2SWtKaGMybGpWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltSmhjMlVpT2pZd0xDSnRZVzUwYVhOellYTWlPbHN4TERJc05Td3hNQ3d4TlN3eU1Dd3pNRjBzSW0xaGVGOXBiblJsY25aaGJDSTZNVGd3TURBd01DNHdMQ0p0YVc1ZmFXNTBaWEoyWVd3aU9qRXdNREF1TUN3aWJuVnRYMjFwYm05eVgzUnBZMnR6SWpvd2ZTd2lhV1FpT2lKbU1UQTBOelE1TkMxaFlURTJMVFE1WTJJdFlXRTRNeTB3TURZeFlqa3dZalV4TURBaUxDSjBlWEJsSWpvaVFXUmhjSFJwZG1WVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWm05eWJXRjBkR1Z5SWpwN0ltbGtJam9pTWpreU5EUmxZekV0WW1ReFlpMDBObVJqTFdFeVpEQXRNakkwTWpjek9UYzBNR1ZtSWl3aWRIbHdaU0k2SWtKaGMybGpWR2xqYTBadmNtMWhkSFJsY2lKOUxDSndiRzkwSWpwN0ltbGtJam9pWmpreFlXUm1abVV0TUdJeVl5MDBOamhpTFdFMk9EZ3ROak00WkRsaFltUXdOemN5SWl3aWMzVmlkSGx3WlNJNklrWnBaM1Z5WlNJc0luUjVjR1VpT2lKUWJHOTBJbjBzSW5ScFkydGxjaUk2ZXlKcFpDSTZJbVUwWkRoaU9UTXlMV0ZtWkdNdE5EYzJZUzFpT0RSa0xUWmlaVEpoTnpWak16Um1OQ0lzSW5SNWNHVWlPaUpDWVhOcFkxUnBZMnRsY2lKOWZTd2lhV1FpT2lJeU9UQXhOVEpqWmkwNE5HWTRMVFF5TjJVdFltTmtPUzB6T1RFeU1ESmhZbVF3TmpNaUxDSjBlWEJsSWpvaVRHbHVaV0Z5UVhocGN5SjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmtZWGx6SWpwYk1Td3hOVjE5TENKcFpDSTZJalJoTm1VNU5HSmlMVEZtWlRJdE5HUmtOUzFpTlRFd0xXRXpZVFZpT0Rrd01UaGpZaUlzSW5SNWNHVWlPaUpFWVhselZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1SaGRHRmZjMjkxY21ObElqcDdJbWxrSWpvaU4yTTJNVFV4WkRFdFptUTJNeTAwWkRNeUxUazRaamd0TW1WalpXUmlaVFkwTVRFeklpd2lkSGx3WlNJNklrTnZiSFZ0YmtSaGRHRlRiM1Z5WTJVaWZTd2laMng1Y0dnaU9uc2lhV1FpT2lJNFpqWm1NamM1T1MxaE9XSmpMVFJsWWpZdE9UVmtZeTB4WkRKaE56a3laalkyTVRjaUxDSjBlWEJsSWpvaVRHbHVaU0o5TENKb2IzWmxjbDluYkhsd2FDSTZiblZzYkN3aWJYVjBaV1JmWjJ4NWNHZ2lPbTUxYkd3c0ltNXZibk5sYkdWamRHbHZibDluYkhsd2FDSTZleUpwWkNJNklqUTJNVFZsTkdJeUxURTFPR010TkdFM1l5MWlOR0UwTFRsak9ETTFOMkkwWWpnMk1pSXNJblI1Y0dVaU9pSk1hVzVsSW4wc0luTmxiR1ZqZEdsdmJsOW5iSGx3YUNJNmJuVnNiSDBzSW1sa0lqb2labVUyT0RneVpEa3RabVUxWkMwMFl6UTJMV0k0TjJNdFpUQm1abUZtWTJFM01tVTVJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWNHeHZkQ0k2Ym5Wc2JDd2lkR1Y0ZENJNklqZzBORE01TnpBaWZTd2lhV1FpT2lKaE9UVmxOelJsWVMwM05XWm1MVFF4TTJFdFlqaGtOeTFsTUdSaU9ETTJOamc1T0dFaUxDSjBlWEJsSWpvaVZHbDBiR1VpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWkdGNWN5STZXekVzT0N3eE5Td3lNbDE5TENKcFpDSTZJak5pTTJKak5tTmtMVEF6TldNdE5EY3dZeTA1T1dFNExURXlaR0poWlRKaVlqUmpaU0lzSW5SNWNHVWlPaUpFWVhselZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3ZlN3aWFXUWlPaUl3TURKbU5URXlNeTA1Tm1ReExUUmhPV0l0WVROaE15MDJOVGRpT0RnNFpqaGhOV0VpTENKMGVYQmxJam9pV1dWaGNuTlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWJXRjRYMmx1ZEdWeWRtRnNJam8xTURBdU1Dd2liblZ0WDIxcGJtOXlYM1JwWTJ0eklqb3dmU3dpYVdRaU9pSm1PRFpqWVRZM09DMHpNMlZtTFRReE5EUXRZakZpTkMwek0yVmtNelk1TTJOaE56a2lMQ0owZVhCbElqb2lRV1JoY0hScGRtVlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVltOTBkRzl0WDNWdWFYUnpJam9pYzJOeVpXVnVJaXdpWm1sc2JGOWhiSEJvWVNJNmV5SjJZV3gxWlNJNk1DNDFmU3dpWm1sc2JGOWpiMnh2Y2lJNmV5SjJZV3gxWlNJNklteHBaMmgwWjNKbGVTSjlMQ0pzWldaMFgzVnVhWFJ6SWpvaWMyTnlaV1Z1SWl3aWJHVjJaV3dpT2lKdmRtVnliR0Y1SWl3aWJHbHVaVjloYkhCb1lTSTZleUoyWVd4MVpTSTZNUzR3ZlN3aWJHbHVaVjlqYjJ4dmNpSTZleUoyWVd4MVpTSTZJbUpzWVdOckluMHNJbXhwYm1WZlpHRnphQ0k2V3pRc05GMHNJbXhwYm1WZmQybGtkR2dpT25zaWRtRnNkV1VpT2pKOUxDSndiRzkwSWpwdWRXeHNMQ0p5Wlc1a1pYSmZiVzlrWlNJNkltTnpjeUlzSW5KcFoyaDBYM1Z1YVhSeklqb2ljMk55WldWdUlpd2lkRzl3WDNWdWFYUnpJam9pYzJOeVpXVnVJbjBzSW1sa0lqb2lNbVZrTnpBNVl6QXRZV1F5TWkwME56YzRMV0ZoWVdVdFpqbGxNemN3TURSbE0yTXdJaXdpZEhsd1pTSTZJa0p2ZUVGdWJtOTBZWFJwYjI0aWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWIzWmxjbXhoZVNJNmV5SnBaQ0k2SWpKbFpEY3dPV013TFdGa01qSXRORGMzT0MxaFlXRmxMV1k1WlRNM01EQTBaVE5qTUNJc0luUjVjR1VpT2lKQ2IzaEJibTV2ZEdGMGFXOXVJbjBzSW5Cc2IzUWlPbnNpYVdRaU9pSm1PVEZoWkdabVpTMHdZakpqTFRRMk9HSXRZVFk0T0MwMk16aGtPV0ZpWkRBM056SWlMQ0p6ZFdKMGVYQmxJam9pUm1sbmRYSmxJaXdpZEhsd1pTSTZJbEJzYjNRaWZYMHNJbWxrSWpvaVlXWTNaV1k1TlRJdE5EZGtNeTAwTWpKakxXRXpNemN0T0dNNU9URmlZelptTldVNElpd2lkSGx3WlNJNklrSnZlRnB2YjIxVWIyOXNJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbU5oYkd4aVlXTnJJanB1ZFd4c0xDSndiRzkwSWpwN0ltbGtJam9pWmpreFlXUm1abVV0TUdJeVl5MDBOamhpTFdFMk9EZ3ROak00WkRsaFltUXdOemN5SWl3aWMzVmlkSGx3WlNJNklrWnBaM1Z5WlNJc0luUjVjR1VpT2lKUWJHOTBJbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lJMFlXVXpPR0V3WXkxa09UZGpMVFJrTkRRdE9HSXlNUzAwTXpnMll6TmpOVEJsWlRRaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFN3aWRHOXZiSFJwY0hNaU9sdGJJazVoYldVaUxDSk9SVU5QUmxOZlIwOU5NMTlHVDFKRlEwRlRWQ0pkTEZzaVFtbGhjeUlzSWpFdU5EWWlYU3hiSWxOcmFXeHNJaXdpTUM0MU9TSmRYWDBzSW1sa0lqb2lNMlJoWkRGbU5tWXRNekpqT1MwME56UXlMVGxqTURRdE4ySTVPVFExWm1Nd1l6WmpJaXdpZEhsd1pTSTZJa2h2ZG1WeVZHOXZiQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZlMzBzSW1sa0lqb2lNamt5TkRSbFl6RXRZbVF4WWkwME5tUmpMV0V5WkRBdE1qSTBNamN6T1RjME1HVm1JaXdpZEhsd1pTSTZJa0poYzJsalZHbGphMFp2Y20xaGRIUmxjaUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzWVdKbGJDSTZleUoyWVd4MVpTSTZJazlpYzJWeWRtRjBhVzl1Y3lKOUxDSnlaVzVrWlhKbGNuTWlPbHQ3SW1sa0lqb2lZalkzTW1GbE5tTXRZVGN5TUMwME56STFMVGxpTkRBdFpEbGpPVFU0WXpRek9EVTRJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZWMTlMQ0pwWkNJNklqWXhOMkU0WWpKaUxUbGtZV0V0TkRWbU55MDRNalZtTFdJNE56ZG1PREJqTXpnMVpTSXNJblI1Y0dVaU9pSk1aV2RsYm1SSmRHVnRJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbU5oYkd4aVlXTnJJanB1ZFd4c0xDSmpiMngxYlc1ZmJtRnRaWE1pT2xzaWVDSXNJbmtpWFN3aVpHRjBZU0k2ZXlKNElqcDdJbDlmYm1SaGNuSmhlVjlmSWpvaVFVRkNRV3BQTmpKa1ZVbEJRVU5xTnpoaVdqRlJaMEZCUlVkeU1YUnVWa05CUVVGQk9HdERNMlJWU1VGQlQyaG5Va3hrTVZGblFVRXdUVGxJZEROV1EwRkJSRUZXTlU4elpGVkpRVUZMYWtkc2NtUXhVV2RCUVd0RVYyRjBNMVpEUVVGRFFYWmxWek5rVlVsQlFVZG5jelppWkRGUlowRkJWVXAyYzNRelZrTWlMQ0prZEhsd1pTSTZJbVpzYjJGME5qUWlMQ0p6YUdGd1pTSTZXekV5WFgwc0lua2lPbnNpWDE5dVpHRnljbUY1WDE4aU9pSkJRVUZCV1V4WlpVZFZRVUZCUVVSQmFHbG5XbEZCUVVGQlFVSllUV2hzUVVGQlFVRlpSRFJMUjJ0QlFVRkJRVUU0ZDNOaFVVRkJRVUZOUTI1RVVuQkJRVUZCUVZsRE5IcEhhMEZCUVVGRVFYTXhOR0ZSUVVGQlFVRkJOV2xvY0VGQlFVRkJXVXMxU0VoclFVRkJRVUpuY210alpWRkJRVUZCUjBOMVVuZzFRU0lzSW1SMGVYQmxJam9pWm14dllYUTJOQ0lzSW5Ob1lYQmxJanBiTVRKZGZYMTlMQ0pwWkNJNklqWmxNVFkwWWpZMUxUaGxNMkl0TkRrME9TMDROREV3TFRrNU4yWTJOek5oTTJVM01TSXNJblI1Y0dVaU9pSkRiMngxYlc1RVlYUmhVMjkxY21ObEluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltTmhiR3hpWVdOcklqcHVkV3hzTENKamIyeDFiVzVmYm1GdFpYTWlPbHNpZUNJc0lua2lYU3dpWkdGMFlTSTZleUo0SWpwN0lsOWZibVJoY25KaGVWOWZJam9pUVVGRVFWWTFUek5rVlVsQlFVdHFSMnh5WkRGUlowRkJhMFJYWVhRelZrTkJRVUkwY0VveU0yUlZTVUZCUjBGVWIySmtNVkZuUVVGVFNVdHJkRE5XUTBGQlFYYzRZV1V6WkZWSlFVRkNhR2R4TjJReFVXZEJRVUZOSzNWME0xWkRRVUZFYjFCaVN6TmtWVWxCUVU1RGMzUmlaREZSWjBGQmRVSjFOWFF6VmtOQlFVTm5hWEo1TTJSVlNVRkJTV28xZGpka01WRm5RVUZqUjJwRWRETldRMEZCUWxreE9HRXpaRlZKUVVGRlFrZDVjbVF4VVdkQlFVdE1XRTUwTTFaRFFVRkJVVXBPUnpOa1ZVbEJRVkJwVXpGTVpERlJaMEZCTkVGSVdYUXpWa05CUVVSSlkwNTFNMlJWU1VGQlRFUm1NM0prTVZGblFVRnRSVGRwZEROV1EwRkJRMEYyWlZjelpGVkpRVUZIWjNNMlltUXhVV2RCUVZWS2RuTjBNMVpEUVVGQk5FTjJRek5rVlVsQlFVTkNOVGczWkRGUlowRkJRMDlxTW5RelZrTkJRVVIzVm5aeE0yUlZTVUZCVG1wR0wySmtNVkZuUVVGM1JGRkNkVWhXUTBGQlEyOXZkMU0wWkZWSlFVRktRVk5EVEdneFVXZEJRV1ZKUlV4MVNGWkRRVUZDWnpoQk5qUmtWVWxCUVVWb1prVnlhREZSWjBGQlRVMDBWblZJVmtOQlFVRlpVRkp0TkdSVlNVRkJRVU56U0V4b01WRm5RVUUyUW05bmRVaFdRMEZCUkZGcFUwODBaRlZKUVVGTWFqUktjbWd4VVdkQlFXOUhZM0YxU0ZaRFFVRkRTVEZwTWpSa1ZVbEJRVWhDUmsxaWFERlJaMEZCVjB4Uk1IVklWa05CUVVKQlNYcHBOR1JWU1VGQlEybFRUemRvTVZGblFVRkZRVVV2ZFVoV1EwRkJSRFJpTUVzMFpGVkpRVUZQUkdWU1ltZ3hVV2RCUVhsRk1VcDFTRlpEUVVGRGQzWkZlVFJrVlVsQlFVcG5jbFZNYURGUlowRkJaMHB3VkhWSVZrTkJRVUp2UTFabE5HUlZTVUZCUmtJMFYzSm9NVkZuUVVGUFQyUmtkVWhXUTBGQlFXZFdiVWMwWkZWSlFVRkJha1phVEdneFVXZEJRVGhFVG05MVNGWkRRVUZFV1c5dGRUUmtWVWxCUVUxQlVtSTNhREZSWjBGQmNVbENlWFZJVmtOQlFVTlJOek5YTkdSVlNVRkJTR2hsWldKb01WRm5RVUZaVFRFNGRVaFdRMEZCUWtsUVNVTTBaRlZKUVVGRVEzSm5OMmd4VVdkQlFVZENjVWgxU0ZaRFFVRkJRV2xaY1RSa1ZVbEJRVTlxTTJwaWFERlJaMEZCTUVkaFVuVklWa05CUVVNME1WcFROR1JWU1VGQlMwSkZiVXhvTVZGblFVRnBURTlpZFVoV1EwRkJRbmRKY0NzMFpGVkpRVUZHYVZKdmNtZ3hVV2RCUVZGQlEyMTFTRlpEUVVGQmIySTJiVFJrVlVsQlFVSkVaWEpNYURGUlowRkJLMFY1ZDNWSVZrTkJRVVJuZFRkUE5HUlZTVUZCVFdkeGREZG9NVkZuUVVGelNtMDJkVWhXUTBGQlExbERURFkwWkZWSlFVRkpRak4zWW1neFVXZEJRV0ZQWWtWMVNGWkRRVUZDVVZaamFUUmtWVWxCUVVScVJYazNhREZSWjBGQlNVUlFVSFZJVmtOQlFVRkpiM1JMTkdSVlNVRkJVRUZSTVhKb01WRm5RVUV5U0M5YWRVaFdRMEZCUkVFM2RIazBaRlZKUVVGTGFHUTBUR2d4VVdkQlFXdE5lbXAxU0ZaRFFVRkNORThyWlRSa1ZVbEJRVWREY1RaeWFERlJaMEZCVTBKdWRYVklWa05CUVVGM2FWQkhOR1JWU1VGQlFtb3pPVXhvTVZGblFVRkJSMkkwZFVoV1EwRkJSRzh4VUhVMFpGVkpRVUZPUWtRdk4yZ3hVV2RCUVhWTVNVTjFXRlpEUVVGRFowbFJZVFZrVlVsQlFVbHBVVU5pYkRGUlowRkJZMUE0VFhWWVZrTkJRVUpaWW1oRE5XUlZTVUZCUlVSa1JUZHNNVkZuUVVGTFJYZFlkVmhXUTBGQlFWRjFlSEUxWkZWSlFVRlFaM0JJY213eFVXZEJRVFJLWjJoMVdGWkRRVUZFU1VKNVZ6VmtWVWxCUVV4Q01rdE1iREZSWjBGQmJVOVZjblZZVmtOQlFVTkJWa01yTldSVlNVRkJSMnBFVFhKc01WRm5RVUZWUkVreWRWaFdRMEZCUVRSdlZHMDFaRlZKUVVGRFFWRlFZbXd4VVdkQlFVTklPVUYxV0ZaRFFVRkVkemRWVHpWa1ZVbEJRVTVvWTFJM2JERlJaMEZCZDAxMFMzVllWa05CUVVOdlQyczJOV1JWU1VGQlNrTndWV0pzTVZGblFVRmxRbWhXZFZoV1EwRkJRbWRvTVdrMVpGVkpRVUZGYWpKWE4yd3hVV2RCUVUxSFZtWjFXRlpEUVVGQldURkhTelZrVlVsQlFVRkNSRnB5YkRGUlowRkJOa3hHY0hWWVZrTkJRVVJSU1VjeU5XUlZTVUZCVEdsUVkweHNNVkZuUVVGdlVEVjZkVmhXUTBGQlEwbGlXR1UxWkZWSlFVRklSR05sY213eFVXZEJRVmRGZEN0MVdGWkRJaXdpWkhSNWNHVWlPaUptYkc5aGREWTBJaXdpYzJoaGNHVWlPbHN4TkRSZGZTd2llU0k2ZXlKZlgyNWtZWEp5WVhsZlh5STZJa0ZCUVVGSlQyaHFTVlZCUVVGQlFVRkxSVGhvVVVGQlFVRkJRbTlQYVVaQlFVRkJRVFJMWTJ4SlZVRkJRVUZCWjNwVFNXaFJRVUZCUVVkRWVVaDVSa0ZCUVVGQmIwSmpaRWxWUVVGQlFVUkJkbEZ2YUZGQlFVRkJRVUpySzBOQ1FVRkJRVUZKUVhKdFNVVkJRVUZCUW1kaUswbG5VVUZCUVVGTFJGVXphVUpCUVVGQlFUUkVibUpKUlVGQlFVRkVaMUIzZDJoUlFVRkJRVUZDUjFCVFJrRkJRVUZCUVVWNGRVbFZRVUZCUVVKQlZXRjNhRkZCUVVGQlNVSlhObWxHUVVGQlFVRjNSbk52U1d0QlFVRkJRMmRGZWxscFVVRkJRVUZKUkV4UmVVcEJRVUZCUVZsSlRsSkphMEZCUVVGRVFWbFViMmxSUVVGQlFVTkNRVWw1U2tGQlFVRkJaMEkwVFVsclFVRkJRVUZuV1hkTmFWRkJRVUZCVFVOdUsybEdRVUZCUVVGWlQzcDRTVlZCUVVGQlEwRTJkR2RvVVVGQlFVRkpSRzkyZVVaQlFVRkJRVzlQWVcxSlZVRkJRVUZEWjJGd1oyaFJRVUZCUVV0RWRXbFRSa0ZCUVVGQmIwaEtOMGxWUVVGQlFVRkJaVEpGYUZGQlFVRkJSVU5FVW5sR1FVRkJRVUZ2U1hOMFNWVkJRVUZCUTBGUlIxVm9VVUZCUVVGSFJERnVRMFpCUVVGQlFWRkxjbFZKVlVGQlFVRkJaMmRSZDJsUlFVRkJRVUZDV1ZKRFNrRkJRVUZCTkVNMU9FbHJRVUZCUVVOQmNscE5hVkZCUVVGQlFVRnpjWGxLUVVGQlFVRnZTM0pEU1d0QlFVRkJSR2RWY2xGcFVVRkJRVUZEUkRkd1UwcEJRVUZCUVZsTFQxaEphMEZCUVVGQ1owNHpkMmxSUVVGQlFVZEVURmxEU2tGQlFVRkJXVVk1UmtsclFVRkJRVUpuVkdwWmFWRkJRVUZCUjBFNVNubEtRVUZCUVVGWlEzZFpTV3RCUVVGQlJFRTFaM2RwVVVGQlFVRkJRMmhCVTBwQlFVRkJRVmxHZGpKSlZVRkJRVUZDUVdwMVkyaFJRVUZCUVVGRVFqSkRSa0ZCUVVGQk5GQlFTa2xWUVVGQlFVUm5OVTEzYUZGQlFVRkJUVVJXZW5sR1FVRkJRVUYzVFdKVFNWVkJRVUZCUVdjcmRsbG9VVUZCUVVGSFFYUkhlVXBCUVVGQlFYZEhRUzlKYTBGQlFVRkNRV2N4YjJsUlFVRkJRVTFEYkdSVFNrRkJRVUZCVVUxcFVVbHJRVUZCUVVSbmNUUnJhVkZCUVVGQlIwTlFaMmxLUVVGQlFVRkJTRTQzU1d0QlFVRkJRa0ZIYmpCcFVVRkJRVUZIUkVKbWFVcEJRVUZCUVc5SGFVRkphMEZCUVVGQ1FUVnVPR2xSUVVGQlFVOUNhbVo1U2tGQlFVRkJaMDlHSzBsclFVRkJRVU5CVEVodmFWRkJRVUZCUzBJelpGTktRVUZCUVVGdlRVcDNTV3RCUVVGQlFXZDVNMFZwVVVGQlFVRkxSRlJqYVVwQlFVRkJRVWxPZUhwSmEwRkJRVUZEWjJ4S1oybFJRVUZCUVVWQ1RuWlRTa0ZCUVVGQmQwRllhVWxyUVVGQlFVSkJORVZqYWxGQlFVRkJTME0yY2xOT1FVRkJRVUZKU2xWVVNrVkJRVUZCUTJkVlZUQnJVVUZCUVVGRlFVOW9lVkpCUVVGQlFYZE5ja0ZLUlVGQlFVRkNRVWhMWjJ0UlFVRkJRVTFDZEdwNVVrRkJRVUZCVVV3NU1rcEZRVUZCUVVOQlUxWlJhMUZCUVVGQlQwUlVUVk5TUVVGQlFVRkpSalJRU2tWQlFVRkJRa0ZwWm5kcVVVRkJRVUZGUXpBMlUwNUJRVUZCUVZsT0wxZEpNRUZCUVVGRVFWYzVXV3BSUVVGQlFVRkVXVEZUVGtGQlFVRkJXVVpVVmtrd1FVRkJRVVJCY2pnNGFsRkJRVUZCUlVGTWVXbE9RVUZCUVVGdlIySkZTVEJCUVVGQlFtZHdUVUZxVVVGQlFVRkZSR2wyUTA1QlFVRkJRVUZEUXpWSk1FRkJRVUZCWnpFM1oycFJRVUZCUVVWRFQzVkRUa0ZCUVVGQldVVlhORWt3UVVGQlFVRm5kbUp6YWxGQlFVRkJUMEV3ZG5sT1FVRkJRVUZ2UzNwRFNUQkJRVUZCUkdkRk4yZHFVVUZCUVVGQlFqZHlVMDVCUVVGQlFWRlBTMmxKTUVGQlFVRkVaM2hoTUdwUlFVRkJRVXREY0hWRFRrRkJRVUZCVVVrelJFa3dRVUZCUVVGblJITkJhbEZCUVVGQlFVTlFka05PUVVGQlFVRTBRU3MxU1RCQlFVRkJRMmRZTjJkcVVVRkJRVUZKUTNaMGVVNUJRVUZCUVZGUUt6SkpNRUZCUVVGQ1oxRmlUV3BSUVVGQlFVdERSSEo1VGtGQlFVRkJkMDFYY2trd1FVRkJRVU5CUldKUmFsRkJRVUZCUTBKa2RrTk9RVUZCUVVFMFMycEZTVEJCUVVGQlJHZFdUMk5xVVVGQlFVRkJRVUpEYVZKQlFVRkJRVUZMTUhOS1JVRkJRVUZCWjJOc1NXdFJRVUZCUVVOQk0yVkRVa0ZCUVVGQlVWQjVaRXBGUVVGQlFVSkJMMG93YTFGQlFVRkJSVVE0YmxOU1FTSXNJbVIwZVhCbElqb2labXh2WVhRMk5DSXNJbk5vWVhCbElqcGJNVFEwWFgxOWZTd2lhV1FpT2lJM00yTmlNemN4T1MwMVlXSmlMVFJtTTJZdE9USmhPQzAyTTJGaU1tUmhNVGs1WkRjaUxDSjBlWEJsSWpvaVEyOXNkVzF1UkdGMFlWTnZkWEpqWlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKa1lYbHpJanBiTVN3MExEY3NNVEFzTVRNc01UWXNNVGtzTWpJc01qVXNNamhkZlN3aWFXUWlPaUkyWTJNMFpEUXhNaTAxWm1RekxUUmxaVFF0T0RNNU9DMDVabUZoTnprMllUTTVPRFFpTENKMGVYQmxJam9pUkdGNWMxUnBZMnRsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKcGRHVnRjeUk2VzNzaWFXUWlPaUk0T0dRd01USTRZUzA0T1RkbExUUTNNalF0T0RJeU5TMWhZMlZoWWpBMVkyWXhNV0lpTENKMGVYQmxJam9pVEdWblpXNWtTWFJsYlNKOUxIc2lhV1FpT2lKak9XWmlPRGN6TnkwME1HUmpMVFEzTldZdE9ESmxaaTB3WlRZM09HTTBNV1k1TUdVaUxDSjBlWEJsSWpvaVRHVm5aVzVrU1hSbGJTSjlMSHNpYVdRaU9pSXdOVE5rWkdJeFpDMDRaalZrTFRRMU56Y3RZVE5pWXkweVpUTTJaRFl3TW1SbU4yVWlMQ0owZVhCbElqb2lUR1ZuWlc1a1NYUmxiU0o5TEhzaWFXUWlPaUkyTVRkaE9HSXlZaTA1WkdGaExUUTFaamN0T0RJMVppMWlPRGMzWmpnd1l6TTROV1VpTENKMGVYQmxJam9pVEdWblpXNWtTWFJsYlNKOVhTd2ljR3h2ZENJNmV5SnBaQ0k2SW1ZNU1XRmtabVpsTFRCaU1tTXRORFk0WWkxaE5qZzRMVFl6T0dRNVlXSmtNRGMzTWlJc0luTjFZblI1Y0dVaU9pSkdhV2QxY21VaUxDSjBlWEJsSWpvaVVHeHZkQ0o5ZlN3aWFXUWlPaUkxT0RjelpUUmlaUzFrWWpZNExUUXdNbVF0T1RWaU1DMDNaVEZrTVRnNFpXRm1abUVpTENKMGVYQmxJam9pVEdWblpXNWtJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhwYm1WZllXeHdhR0VpT25zaWRtRnNkV1VpT2pBdU1YMHNJbXhwYm1WZlkyRndJam9pY205MWJtUWlMQ0pzYVc1bFgyTnZiRzl5SWpwN0luWmhiSFZsSWpvaUl6Rm1OemRpTkNKOUxDSnNhVzVsWDJwdmFXNGlPaUp5YjNWdVpDSXNJbXhwYm1WZmQybGtkR2dpT25zaWRtRnNkV1VpT2pWOUxDSjRJanA3SW1acFpXeGtJam9pZUNKOUxDSjVJanA3SW1acFpXeGtJam9pZVNKOWZTd2lhV1FpT2lKa1lqSXpZbU5qTkMxak9HWTFMVFF4TXpVdFltTXpaUzAxT0dVMlpERXpNelJtTlRnaUxDSjBlWEJsSWpvaVRHbHVaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUp0YjI1MGFITWlPbHN3TERJc05DdzJMRGdzTVRCZGZTd2lhV1FpT2lKbVlUZzNaakZqTXkwNU0yRTNMVFF3TlRNdFltRXpaUzAyTW1Vd09USTNNall6TkdNaUxDSjBlWEJsSWpvaVRXOXVkR2h6VkdsamEyVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbTF2Ym5Sb2N5STZXekFzTVN3eUxETXNOQ3cxTERZc055dzRMRGtzTVRBc01URmRmU3dpYVdRaU9pSTBaV1F4WVdNek9TMWpZV05oTFRRM1pEQXRZak15TkMxaVl6bG1aRE13TmpVMVlqWWlMQ0owZVhCbElqb2lUVzl1ZEdoelZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1SaGVYTWlPbHN4TERJc015dzBMRFVzTml3M0xEZ3NPU3d4TUN3eE1Td3hNaXd4TXl3eE5Dd3hOU3d4Tml3eE55d3hPQ3d4T1N3eU1Dd3lNU3d5TWl3eU15d3lOQ3d5TlN3eU5pd3lOeXd5T0N3eU9Td3pNQ3d6TVYxOUxDSnBaQ0k2SW1SaU5EQTFNRFZtTFdVNE4yTXRORGd3WWkxaE5UY3pMV0UyWldZd05qWXlPVEV5TnlJc0luUjVjR1VpT2lKRVlYbHpWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteGhZbVZzSWpwN0luWmhiSFZsSWpvaVRrVkRUMFpUWDBkUFRUTWlmU3dpY21WdVpHVnlaWEp6SWpwYmV5SnBaQ0k2SWpSaFpUTTRZVEJqTFdRNU4yTXROR1EwTkMwNFlqSXhMVFF6T0Raak0yTTFNR1ZsTkNJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjFkZlN3aWFXUWlPaUl3TlROa1pHSXhaQzA0WmpWa0xUUTFOemN0WVROaVl5MHlaVE0yWkRZd01tUm1OMlVpTENKMGVYQmxJam9pVEdWblpXNWtTWFJsYlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKc1lXSmxiQ0k2ZXlKMllXeDFaU0k2SWs1RlEwOUdVMTlOWVhOelFtRjVJbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lKbVpUWTRPREprT1MxbVpUVmtMVFJqTkRZdFlqZzNZeTFsTUdabVlXWmpZVGN5WlRraUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFgwc0ltbGtJam9pWXpsbVlqZzNNemN0TkRCa1l5MDBOelZtTFRneVpXWXRNR1UyTnpoak5ERm1PVEJsSWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWJHRmlaV3dpT25zaWRtRnNkV1VpT2lKSE1WOVRVMVJmUjB4UFFrRk1JbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lJelltUXlNVEpoTXkweE5ERTRMVFEzTW1RdE9HRm1OaTA1WkdabE1UY3haalpoTURjaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFgwc0ltbGtJam9pT0Roa01ERXlPR0V0T0RrM1pTMDBOekkwTFRneU1qVXRZV05sWVdJd05XTm1NVEZpSWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW1OdmJIVnRibDl1WVcxbGN5STZXeUo0SWl3aWVTSmRMQ0prWVhSaElqcDdJbmdpT25zaVgxOXVaR0Z5Y21GNVgxOGlPaUpCUVVKQmFrODJNbVJWU1VGQlEybzNPR0phTVZGblFVRkZSM0l4ZEc1V1EwRkJSRFF5VUdreVpGVkpRVUZQUWtndlRGb3hVV2RCUVhsTVlpOTBibFpEUVVGRGQwcFJUek5rVlVsQlFVcHBWVUp5WkRGUlowRkJaMEZOUzNRelZrTkJRVUp2WTJjeU0yUlZTVUZCUmtSb1JVeGtNVkZuUVVGUFJrRlZkRE5XUTBGQlFXZDJlR1V6WkZWSlFVRkJaM1ZITjJReFVXZEJRVGhLZDJWME0xWkRRVUZFV1VONVN6TmtWVWxCUVUxQ05rcGlaREZSWjBGQmNVOXJiM1F6VmtOQlFVTlJWME41TTJSVlNVRkJTR3BJVERka01WRm5RVUZaUkZsNmRETldRMEZCUWtsd1ZHRXpaRlZKUVVGRVFWVlBjbVF4VVdkQlFVZEpUVGwwTTFaRFFVRkJRVGhyUXpOa1ZVbEJRVTlvWjFKTVpERlJaMEZCTUUwNVNIUXpWa05CUVVNMFVHdDFNMlJWU1VGQlMwTjBWSEprTVZGblFVRnBRbmhUZEROV1EwRkJRbmRwTVZjelpGVkpRVUZHYWpaWFRHUXhVV2RCUVZGSGJHTjBNMVpEUVVGQmJ6SkdLek5rVlVsQlFVSkNTRmszWkRGUlowRkJLMHhXYlhRelZrTkJRVVJuU2tkeE0yUlZTVUZCVFdsVVltSmtNVkZuUVVGelFVcDRkRE5XUTBGQlExbGpXRk16WkZWSlFVRkpSR2RrTjJReFVXZEJRV0ZGT1RkME0xWkRRVUZDVVhadU5qTmtWVWxCUVVSbmRHZHlaREZSWjBGQlNVcDVSblF6VmtOQlFVRkpRelJ0TTJSVlNVRkJVRUkxYWt4a01WRm5RVUV5VDJsUWRETldRMEZCUkVGV05VOHpaRlZKUVVGTGFrZHNjbVF4VVdkQlFXdEVWMkYwTTFaRFFVRkNOSEJLTWpOa1ZVbEJRVWRCVkc5aVpERlJaMEZCVTBsTGEzUXpWa05CUVVGM09HRmxNMlJWU1VGQlFtaG5jVGRrTVZGblFVRkJUU3QxZEROV1EwRkJSRzlRWWtzelpGVkpRVUZPUTNOMFltUXhVV2RCUVhWQ2RUVjBNMVpEUVVGRFoybHllVE5rVlVsQlFVbHFOWFkzWkRGUlowRkJZMGRxUkhRelZrTkJRVUpaTVRoaE0yUlZTVUZCUlVKSGVYSmtNVkZuUVVGTFRGaE9kRE5XUTBGQlFWRktUa2N6WkZWSlFVRlFhVk14VEdReFVXZEJRVFJCU0ZsME0xWkRRVUZFU1dOT2RUTmtWVWxCUVV4RVpqTnlaREZSWjBGQmJVVTNhWFF6VmtOQlFVTkJkbVZYTTJSVlNVRkJSMmR6Tm1Ka01WRm5RVUZWU25aemRETldRMEZCUVRSRGRrTXpaRlZKUVVGRFFqVTROMlF4VVdkQlFVTlBhakowTTFaRFFVRkVkMVoyY1ROa1ZVbEJRVTVxUmk5aVpERlJaMEZCZDBSUlFuVklWa05CUVVOdmIzZFROR1JWU1VGQlNrRlRRMHhvTVZGblFVRmxTVVZNZFVoV1EwRkJRbWM0UVRZMFpGVkpRVUZGYUdaRmNtZ3hVV2RCUVUxTk5GWjFTRlpEUVVGQldWQlNiVFJrVlVsQlFVRkRjMGhNYURGUlowRkJOa0p2WjNWSVZrTkJRVVJSYVZOUE5HUlZTVUZCVEdvMFNuSm9NVkZuUVVGdlIyTnhkVWhXUTBGQlEwa3hhVEkwWkZWSlFVRklRa1pOWW1neFVXZEJRVmRNVVRCMVNGWkRRVUZDUVVsNmFUUmtWVWxCUVVOcFUwODNhREZSWjBGQlJVRkZMM1ZJVmtOQlFVUTBZakJMTkdSVlNVRkJUMFJsVW1Kb01WRm5RVUY1UlRGS2RVaFdRMEZCUTNkMlJYazBaRlZKUVVGS1ozSlZUR2d4VVdkQlFXZEtjRlIxU0ZaRFFVRkNiME5XWlRSa1ZVbEJRVVpDTkZkeWFERlJaMEZCVDA5a1pIVklWa05CUVVGblZtMUhOR1JWU1VGQlFXcEdXa3hvTVZGblFVRTRSRTV2ZFVoV1EwRkJSRmx2YlhVMFpGVkpRVUZOUVZKaU4yZ3hVV2RCUVhGSlFubDFTRlpEUVVGRFVUY3pWelJrVlVsQlFVaG9aV1ZpYURGUlowRkJXVTB4T0hWSVZrTkJRVUpKVUVsRE5HUlZTVUZCUkVOeVp6ZG9NVkZuUVVGSFFuRklkVWhXUTBGQlFVRnBXWEUwWkZWSlFVRlBhak5xWW1neFVXZEJRVEJIWVZKMVNGWkRRVUZETkRGYVV6UmtWVWxCUVV0Q1JXMU1hREZSWjBGQmFVeFBZblZJVmtOQlFVSjNTWEFyTkdSVlNVRkJSbWxTYjNKb01WRm5RVUZSUVVOdGRVaFdRMEZCUVc5aU5tMDBaRlZKUVVGQ1JHVnlUR2d4VVdkQlFTdEZlWGQxU0ZaRFFVRkVaM1UzVHpSa1ZVbEJRVTFuY1hRM2FERlJaMEZCYzBwdE5uVklWa05CUVVOWlEwdzJOR1JWU1VGQlNVSXpkMkpvTVZGblFVRmhUMkpGZFVoV1EwRkJRbEZXWTJrMFpGVkpRVUZFYWtWNU4yZ3hVV2RCUVVsRVVGQjFTRlpESWl3aVpIUjVjR1VpT2lKbWJHOWhkRFkwSWl3aWMyaGhjR1VpT2xzeE5ERmRmU3dpZVNJNmV5SmZYMjVrWVhKeVlYbGZYeUk2SW5wamVrMTZUWHBOUjJ0RVRucE5lazE2VFhkalVVMHpUWHBOZWsxNlFtaEJUWHBOZWsxNlRYcEdNRUY2VFhwTmVrMTZUVmhSUkUxNlRYcE5lazE0WkVGdGNHMWFiVnB0V2tZd1FVRkJRVUZCUVVGQldWRkhXbTFhYlZwdFdtaG9RVnB0V20xYWJWcHRSMFZFVG5wTmVrMTZUWGRaVVUwelRYcE5lazE2UW5CQlFVRkJRVUZCUVVGSVJVRkJRVUZCUVVGQlFXTlJSMXB0V20xYWJWcG9lRUY2WTNwTmVrMTZUVWRGUW0xYWJWcHRXbTFaV1ZGQlFVRkJRVUZCUVVKb1FYcGplazE2VFhwTlJtdEJlazE2VFhwTmVrMVlVVXB4V20xYWJWcHRVbVJCV20xYWJWcHRXbTFIUlVST2VrMTZUWHBOZDFsUlFVRkJRVUZCUVVGQ2NFRmFiVnB0V20xYWJVaEZRVUZCUVVGQlFVRkJZMUZCUVVGQlFVRkJRVUo0UVUxNlRYcE5lazE2UjFWRVRucE5lazE2VFhkWlVVRkJRVUZCUVVGQlFtaEJiWEJ0V20xYWJWcEdNRU5oYlZwdFdtMWFhMWhSVFROTmVrMTZUWHBDYUVGTmVrMTZUWHBOZWtkVlEyRnRXbTFhYlZwcldsRkhXbTFhYlZwdFdtaHdRVUZCUVVGQlFVRkJTRVZCUVVGQlFVRkJRVUZqVVVGQlFVRkJRVUZCUWpWQlFVRkJRVUZCUVVGSVJVRkJRVUZCUVVGQlFXRlJSRTE2VFhwTmVrMTRiRUZCUVVGQlFVRkJRVWRGUVVGQlFVRkJRVUZCV1ZGQlFVRkJRVUZCUVVKb1FWcHRXbTFhYlZwdFIwVkJlazE2VFhwTmVrMWFVVXB4V20xYWJWcHRVbXhCZW1ONlRYcE5lazFIYTBGNlRYcE5lazE2VFdSUlNuRmFiVnB0V20xU2RFRjZZM3BOZWsxNlRVZHJSRTU2VFhwTmVrMTNZMUZOTTAxNlRYcE5la0pvUVZwdFdtMWFiVnB0UjBWQ2JWcHRXbTFhYlZsWlVVMHpUWHBOZWsxNlFtaEJlbU42VFhwTmVrMUhSVVJPZWsxNlRYcE5kMWxSUkUxNlRYcE5lazE0ZEVGTmVrMTZUWHBOZWtoVlEyRnRXbTFhYlZwcllsRk5NMDE2VFhwTmVrSndRVnB0V20xYWJWcHRSMnRDYlZwdFdtMWFiVmxqVVVweFdtMWFiVnB0VW14QmJYQnRXbTFhYlZwSFZVTmhiVnB0V20xYWExcFJRVUZCUVVGQlFVRkNjRUZCUVVGQlFVRkJRVWRyUVVGQlFVRkJRVUZCWVZGS2NWcHRXbTFhYlZKMFFWcHRXbTFhYlZwdFNFVkNiVnB0V20xYWJWbGxVVXB4V20xYWJWcHRVakZCV20xYWJWcHRXbTFJYTBST2VrMTZUWHBOZDJkUlFVRkJRVUZCUVVGQ05VRmFiVnB0V20xYWJVaHJSRTU2VFhwTmVrMTNZMUZOTTAxNlRYcE5la0p3UVcxd2JWcHRXbTFhUnpCRFlXMWFiVnB0V210aVVVRkJRVUZCUVVGQlFuaEJRVUZCUVVGQlFVRklSVUY2VFhwTmVrMTZUV1JSVFROTmVrMTZUWHBDTlVGNlkzcE5lazE2VFVsRlFVRkJRVUZCUVVGQmFGRkJRVUZCUVVGQlFVTkdRVzF3YlZwdFdtMWFTVVZDYlZwdFdtMWFiVmxqVVVkYWJWcHRXbTFhYUhoQmVtTjZUWHBOZWsxSVJVRkJRVUZCUVVGQlFXVlJRVUZCUVVGQlFVRkNOVUZhYlZwdFdtMWFiVWhyUkU1NlRYcE5lazEzWlZGRVRYcE5lazE2VFhnNVFVRkJRVUZCUVVGQlNVVkJlazE2VFhwTmVrMW9VVWRhYlZwdFdtMWFhVUpCYlhCdFdtMWFiVnBJTUVKdFdtMWFiVnB0V1dWUlIxcHRXbTFhYlZwb05VRk5lazE2VFhwTmVrZ3dRMkZ0V20xYWJWcHJabEZCUVVGQlFVRkJRVU5DUVcxd2JWcHRXbTFhU0RCRFlXMWFiVnB0V210bVVVcHhXbTFhYlZwdFVqbEJXbTFhYlZwdFdtMUpSVUp0V20xYWJWcHRXV2RSVFROTmVrMTZUWHBEUWtGQlFVRkJRVUZCUVVsRlFVRkJRVUZCUVVGQloxRkVUWHBOZWsxNlRYbENRVTE2VFhwTmVrMTZTREJCZWsxNlRYcE5lazFtVVVGQlFVRkJRVUZCUTBKQlFVRkJRVUZCUVVGSlJVSnRXbTFhYlZwdFdXZFJSMXB0V20xYWJWcHBRa0Y2WTNwTmVrMTZUVWxGUVhwTmVrMTZUWHBOYUZGSFdtMWFiVnB0V21sR1FXMXdiVnB0V20xYVNWVkNiVnB0V20xYWJWbG9VVUZCUVVGQlFVRkJRMEpCYlhCdFdtMWFiVnBJTUVGQlFVRkJRVUZCUVdkUlNuRmFiVnB0V20xU09VRkJRVUZCUVVGQlFVbEZRbTFhYlZwdFdtMVpaMUZLY1ZwdFdtMWFiVk5DUVcxd2JWcHRXbTFhU1VWRFlXMWFiVnB0V210b1VVRkJRVUZCUVVGQlEwWkJiWEJ0V20xYWJWcEpWVVJPZWsxNlRYcE5kMmRSUjFwdFdtMWFiVnBwUWtFaUxDSmtkSGx3WlNJNkltWnNiMkYwTmpRaUxDSnphR0Z3WlNJNld6RTBNVjE5Zlgwc0ltbGtJam9pTXpoaU16ZGhZamt0T0dVNFlpMDBaR1l3TFdJMk56RXRNbVF6TmpObE9EZG1ZV0ptSWl3aWRIbHdaU0k2SWtOdmJIVnRia1JoZEdGVGIzVnlZMlVpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdsdVpWOWhiSEJvWVNJNmV5SjJZV3gxWlNJNk1DNHhmU3dpYkdsdVpWOWpZWEFpT2lKeWIzVnVaQ0lzSW14cGJtVmZZMjlzYjNJaU9uc2lkbUZzZFdVaU9pSWpNV1kzTjJJMEluMHNJbXhwYm1WZmFtOXBiaUk2SW5KdmRXNWtJaXdpYkdsdVpWOTNhV1IwYUNJNmV5SjJZV3gxWlNJNk5YMHNJbmdpT25zaVptbGxiR1FpT2lKNEluMHNJbmtpT25zaVptbGxiR1FpT2lKNUluMTlMQ0pwWkNJNklqUmhORFEwT1RNMUxUSm1aVFV0TkdZNVl5MWlOamc0TFRSaU1HRm1ZMlZtT0RjNU55SXNJblI1Y0dVaU9pSk1hVzVsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1ScGJXVnVjMmx2YmlJNk1Td2ljR3h2ZENJNmV5SnBaQ0k2SW1ZNU1XRmtabVpsTFRCaU1tTXRORFk0WWkxaE5qZzRMVFl6T0dRNVlXSmtNRGMzTWlJc0luTjFZblI1Y0dVaU9pSkdhV2QxY21VaUxDSjBlWEJsSWpvaVVHeHZkQ0o5TENKMGFXTnJaWElpT25zaWFXUWlPaUpsTkdRNFlqa3pNaTFoWm1SakxUUTNObUV0WWpnMFpDMDJZbVV5WVRjMVl6TTBaalFpTENKMGVYQmxJam9pUW1GemFXTlVhV05yWlhJaWZYMHNJbWxrSWpvaU5USXpaRFZtWlRFdE5qa3lOUzAwWWpFMExUbGxZbU10TTJFelpXSXlPVGhsT0RFM0lpd2lkSGx3WlNJNklrZHlhV1FpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdsdVpWOWpZWEFpT2lKeWIzVnVaQ0lzSW14cGJtVmZZMjlzYjNJaU9uc2lkbUZzZFdVaU9pSWpabVZsTURoaUluMHNJbXhwYm1WZmFtOXBiaUk2SW5KdmRXNWtJaXdpYkdsdVpWOTNhV1IwYUNJNmV5SjJZV3gxWlNJNk5YMHNJbmdpT25zaVptbGxiR1FpT2lKNEluMHNJbmtpT25zaVptbGxiR1FpT2lKNUluMTlMQ0pwWkNJNklqa3pabVZsTVRCa0xUSTVNR0l0TkRrME9DMDRNalZpTFdZMk5UQmtNMlJrTWpoaU5pSXNJblI1Y0dVaU9pSk1hVzVsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1OaGJHeGlZV05ySWpwdWRXeHNMQ0p3Ykc5MElqcDdJbWxrSWpvaVpqa3hZV1JtWm1VdE1HSXlZeTAwTmpoaUxXRTJPRGd0TmpNNFpEbGhZbVF3TnpjeUlpd2ljM1ZpZEhsd1pTSTZJa1pwWjNWeVpTSXNJblI1Y0dVaU9pSlFiRzkwSW4wc0luSmxibVJsY21WeWN5STZXM3NpYVdRaU9pSm1aVFk0T0RKa09TMW1aVFZrTFRSak5EWXRZamczWXkxbE1HWm1ZV1pqWVRjeVpUa2lMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOVhTd2lkRzl2YkhScGNITWlPbHRiSWs1aGJXVWlMQ0pPUlVOUFJsTmZSbFpEVDAxZlQwTkZRVTVmVFVGVFUwSkJXVjlHVDFKRlEwRlRWQ0pkTEZzaVFtbGhjeUlzSWpBdU9UVWlYU3hiSWxOcmFXeHNJaXdpTUM0NU9DSmRYWDBzSW1sa0lqb2laRFkyTURJd1ltWXRNelppT1MwME1URTRMVGxtWVRRdE5ERmtaamMyWXpJeE5qTmtJaXdpZEhsd1pTSTZJa2h2ZG1WeVZHOXZiQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUp3Ykc5MElqcDdJbWxrSWpvaVpqa3hZV1JtWm1VdE1HSXlZeTAwTmpoaUxXRTJPRGd0TmpNNFpEbGhZbVF3TnpjeUlpd2ljM1ZpZEhsd1pTSTZJa1pwWjNWeVpTSXNJblI1Y0dVaU9pSlFiRzkwSW4wc0luUnBZMnRsY2lJNmV5SnBaQ0k2SWpjeFlqRTRPR0kyTFRjelptRXROREprWXkwNU9ESmxMVGRsTmpJME5URXlObUkyTmlJc0luUjVjR1VpT2lKRVlYUmxkR2x0WlZScFkydGxjaUo5ZlN3aWFXUWlPaUl3TTJNeE5UaGtOUzAxTXpnekxUUXhOVE10WVRNMVpDMDNPRFZqWkRObFpEWTJNbUVpTENKMGVYQmxJam9pUjNKcFpDSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmtZWFJoWDNOdmRYSmpaU0k2ZXlKcFpDSTZJak00WWpNM1lXSTVMVGhsT0dJdE5HUm1NQzFpTmpjeExUSmtNell6WlRnM1ptRmlaaUlzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjBzSW1kc2VYQm9JanA3SW1sa0lqb2lPVE5tWldVeE1HUXRNamt3WWkwME9UUTRMVGd5TldJdFpqWTFNR1F6WkdReU9HSTJJaXdpZEhsd1pTSTZJa3hwYm1VaWZTd2lhRzkyWlhKZloyeDVjR2dpT201MWJHd3NJbTExZEdWa1gyZHNlWEJvSWpwdWRXeHNMQ0p1YjI1elpXeGxZM1JwYjI1ZloyeDVjR2dpT25zaWFXUWlPaUkwWVRRME5Ea3pOUzB5Wm1VMUxUUm1PV010WWpZNE9DMDBZakJoWm1ObFpqZzNPVGNpTENKMGVYQmxJam9pVEdsdVpTSjlMQ0p6Wld4bFkzUnBiMjVmWjJ4NWNHZ2lPbTUxYkd4OUxDSnBaQ0k2SW1JMk56SmhaVFpqTFdFM01qQXRORGN5TlMwNVlqUXdMV1E1WXprMU9HTTBNemcxT0NJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbUpsYkc5M0lqcGJleUpwWkNJNklqZGlOMlk1T1RRMExURTBNR1F0TkdZek1TMWhNR1V4TFdRM09EWmlZMlEzTkRGaFppSXNJblI1Y0dVaU9pSkVZWFJsZEdsdFpVRjRhWE1pZlYwc0lteGxablFpT2x0N0ltbGtJam9pTWprd01UVXlZMll0T0RSbU9DMDBNamRsTFdKalpEa3RNemt4TWpBeVlXSmtNRFl6SWl3aWRIbHdaU0k2SWt4cGJtVmhja0Y0YVhNaWZWMHNJbkJzYjNSZmFHVnBaMmgwSWpveU5UQXNJbkJzYjNSZmQybGtkR2dpT2pjMU1Dd2ljbVZ1WkdWeVpYSnpJanBiZXlKcFpDSTZJamRpTjJZNU9UUTBMVEUwTUdRdE5HWXpNUzFoTUdVeExXUTNPRFppWTJRM05ERmhaaUlzSW5SNWNHVWlPaUpFWVhSbGRHbHRaVUY0YVhNaWZTeDdJbWxrSWpvaU1ETmpNVFU0WkRVdE5UTTRNeTAwTVRVekxXRXpOV1F0TnpnMVkyUXpaV1EyTmpKaElpd2lkSGx3WlNJNklrZHlhV1FpZlN4N0ltbGtJam9pTWprd01UVXlZMll0T0RSbU9DMDBNamRsTFdKalpEa3RNemt4TWpBeVlXSmtNRFl6SWl3aWRIbHdaU0k2SWt4cGJtVmhja0Y0YVhNaWZTeDdJbWxrSWpvaU5USXpaRFZtWlRFdE5qa3lOUzAwWWpFMExUbGxZbU10TTJFelpXSXlPVGhsT0RFM0lpd2lkSGx3WlNJNklrZHlhV1FpZlN4N0ltbGtJam9pTW1Wa056QTVZekF0WVdReU1pMDBOemM0TFdGaFlXVXRaamxsTXpjd01EUmxNMk13SWl3aWRIbHdaU0k2SWtKdmVFRnVibTkwWVhScGIyNGlmU3g3SW1sa0lqb2lOVGczTTJVMFltVXRaR0kyT0MwME1ESmtMVGsxWWpBdE4yVXhaREU0T0dWaFptWmhJaXdpZEhsd1pTSTZJa3hsWjJWdVpDSjlMSHNpYVdRaU9pSXpZbVF5TVRKaE15MHhOREU0TFRRM01tUXRPR0ZtTmkwNVpHWmxNVGN4WmpaaE1EY2lMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOUxIc2lhV1FpT2lKbVpUWTRPREprT1MxbVpUVmtMVFJqTkRZdFlqZzNZeTFsTUdabVlXWmpZVGN5WlRraUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5TEhzaWFXUWlPaUkwWVdVek9HRXdZeTFrT1RkakxUUmtORFF0T0dJeU1TMDBNemcyWXpOak5UQmxaVFFpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlMSHNpYVdRaU9pSmlOamN5WVdVMll5MWhOekl3TFRRM01qVXRPV0kwTUMxa09XTTVOVGhqTkRNNE5UZ2lMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOVhTd2lkR2wwYkdVaU9uc2lhV1FpT2lKaE9UVmxOelJsWVMwM05XWm1MVFF4TTJFdFlqaGtOeTFsTUdSaU9ETTJOamc1T0dFaUxDSjBlWEJsSWpvaVZHbDBiR1VpZlN3aWRHOXZiRjlsZG1WdWRITWlPbnNpYVdRaU9pSm1OelV3WVdSbE55MDVabVpsTFRRNVkyTXRPREkzWkMxalpETTFaRFkxWmpZMU9UVWlMQ0owZVhCbElqb2lWRzl2YkVWMlpXNTBjeUo5TENKMGIyOXNZbUZ5SWpwN0ltbGtJam9pTXpJeE5tVm1OekV0T1Rjd01pMDBOV0U1TFRrd1lqVXRNVEptT0dGaE56QTJOelE1SWl3aWRIbHdaU0k2SWxSdmIyeGlZWElpZlN3aWRHOXZiR0poY2w5c2IyTmhkR2x2YmlJNkltRmliM1psSWl3aWVGOXlZVzVuWlNJNmV5SnBaQ0k2SW1KalpUZzVZMlpoTFdKalpUZ3RORE5tWlMxaE5qUmlMVEUyWWpWaE1UZGpZMkZqTkNJc0luUjVjR1VpT2lKRVlYUmhVbUZ1WjJVeFpDSjlMQ0o1WDNKaGJtZGxJanA3SW1sa0lqb2lZMk0wTlRReFlUVXRZMkkwWlMwME4yTmlMV0kwTlRJdE1XSTNZVGcwTXpoaU1qSm1JaXdpZEhsd1pTSTZJa1JoZEdGU1lXNW5aVEZrSW4xOUxDSnBaQ0k2SW1ZNU1XRmtabVpsTFRCaU1tTXRORFk0WWkxaE5qZzRMVFl6T0dRNVlXSmtNRGMzTWlJc0luTjFZblI1Y0dVaU9pSkdhV2QxY21VaUxDSjBlWEJsSWpvaVVHeHZkQ0o5WFN3aWNtOXZkRjlwWkhNaU9sc2laamt4WVdSbVptVXRNR0l5WXkwME5qaGlMV0UyT0RndE5qTTRaRGxoWW1Rd056Y3lJbDE5TENKMGFYUnNaU0k2SWtKdmEyVm9JRUZ3Y0d4cFkyRjBhVzl1SWl3aWRtVnljMmx2YmlJNklqQXVNVEl1TlNKOWZUc0tJQ0FnSUNBZ0lDQWdJQ0FnSUNCMllYSWdjbVZ1WkdWeVgybDBaVzF6SUQwZ1czc2laRzlqYVdRaU9pSTBOek0yTm1NMFlTMHhNV0ppTFRRM05UQXRPREZtTlMweFpqUXpOalUwWmpFell6Z2lMQ0psYkdWdFpXNTBhV1FpT2lJMk1UZGhPREZoWmkxak5UZzBMVFExTURrdE9HUTNOaTB5Tm1KaU5UQTJZVGRtWTJVaUxDSnRiMlJsYkdsa0lqb2laamt4WVdSbVptVXRNR0l5WXkwME5qaGlMV0UyT0RndE5qTTRaRGxoWW1Rd056Y3lJbjFkT3dvZ0lDQWdJQ0FnSUNBZ0lDQWdJQW9nSUNBZ0lDQWdJQ0FnSUNBZ0lFSnZhMlZvTG1WdFltVmtMbVZ0WW1Wa1gybDBaVzF6S0dSdlkzTmZhbk52Yml3Z2NtVnVaR1Z5WDJsMFpXMXpLVHNLSUNBZ0lDQWdJQ0FnSUNBZ2ZTazdDaUFnSUNBZ0lDQWdJQ0I5T3dvZ0lDQWdJQ0FnSUNBZ2FXWWdLR1J2WTNWdFpXNTBMbkpsWVdSNVUzUmhkR1VnSVQwZ0lteHZZV1JwYm1jaUtTQm1iaWdwT3dvZ0lDQWdJQ0FnSUNBZ1pXeHpaU0JrYjJOMWJXVnVkQzVoWkdSRmRtVnVkRXhwYzNSbGJtVnlLQ0pFVDAxRGIyNTBaVzUwVEc5aFpHVmtJaXdnWm00cE93b2dJQ0FnSUNBZ0lIMHBLQ2s3Q2lBZ0lDQWdJQ0FnQ2lBZ0lDQWdJQ0FnUEM5elkzSnBjSFErQ2lBZ0lDQThMMkp2WkhrK0Nqd3ZhSFJ0YkQ0PSIgd2lkdGg9Ijc5MCIgc3R5bGU9ImJvcmRlcjpub25lICFpbXBvcnRhbnQ7IiBoZWlnaHQ9IjMzMCI+PC9pZnJhbWU+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF84Y2U4NTM0ZDU0MDI0YTBlYmUzMGY2M2M5MWFiZDJkNy5zZXRDb250ZW50KGlfZnJhbWVfMjEwYWIyNDVhNmQxNGYwOTk3MWFiMmVjMTliZjQ0NzQpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9lM2RhM2NkOTJmYzQ0MGYzOGFhZmZhMTI3NDkzZmE4ZC5iaW5kUG9wdXAocG9wdXBfOGNlODUzNGQ1NDAyNGEwZWJlMzBmNjNjOTFhYmQyZDcpOwoKICAgICAgICAgICAgCiAgICAgICAgCjwvc2NyaXB0Pg==" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>



Now we can navigate the map and click on the markers to explorer our findings.

The green markers locate the observations locations. They pop-up an interactive plot with the time-series and scores for the models (hover over the lines to se the scores). The blue markers indicate the nearest model grid point found for the comparison.

<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2016-12-22-boston_light_swim.ipynb)
this notebook, or see a static view [here](http://nbviewer.ipython.org/urls/raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2016-12-22-boston_light_swim.ipynb).
