---
layout: notebook
title: ""
---


# How to search the IOOS CSW catalog with Python tools


This notebook demonstrates a how to query a [Catalog Service for the Web (CSW)](https://en.wikipedia.org/wiki/Catalog_Service_for_the_Web), like the IOOS Catalog, and to parse its results into endpoints that can be used to access the data.

(This notebook uses a custom `ioos_tools` module that needs to be added to the path separately. We recommend cloning the [repository](https://github.com/ioos/notebooks_demos) on GitHub which already includes the most update version of `ioos_tools`.)

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
import os
import sys

ioos_tools = os.path.join(os.path.pardir)
sys.path.append(ioos_tools)
```

Let's start by creating the search filters.
The filter used here constraints the search on a certain geographical region (bounding box), a time span (last week), and some [CF](http://cfconventions.org/Data/cf-standard-names/37/build/cf-standard-name-table.html) variable standard names that represent sea surface temperature.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
from datetime import datetime, timedelta

# Region: Northwest coast.
bbox = [-127, 43, -123.75, 48]
min_lon, max_lon = -127, -123.75
min_lat, max_lat = 43, 48

bbox = [min_lon, min_lat, max_lon, max_lat]
crs = 'urn:ogc:def:crs:OGC:1.3:CRS84'

# Temporal range: Last week.
now = datetime.utcnow()
start, stop = now - timedelta(days=(7)), now

# Sea surface temperature CF names.
cf_names = ['sea_water_temperature',
            'sea_surface_temperature',
            'sea_water_potential_temperature',
            'equivalent_potential_temperature',
            'sea_water_conservative_temperature',
            'pseudo_equivalent_potential_temperature']
```

With these 3 elements it is possible to assemble a [OGC Filter Encoding (FE)](http://www.opengeospatial.org/standards/filter) using the `owslib.fes`\* module.

\* OWSLib is a Python package for client programming with Open Geospatial Consortium (OGC) web service (hence OWS) interface standards, and their related content models.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
from owslib import fes
from ioos_tools.ioos import fes_date_filter

kw = dict(wildCard='*', escapeChar='\\',
          singleChar='?', propertyname='apiso:AnyText')

or_filt = fes.Or([fes.PropertyIsLike(literal=('*%s*' % val), **kw)
                  for val in cf_names])

begin, end = fes_date_filter(start, stop)
bbox_crs = fes.BBox(bbox, crs=crs)

filter_list = [
    fes.And(
        [
            bbox_crs,  # bounding box
            begin, end,  # start and end date
            or_filt  # or conditions (CF variable names)
        ]
    )
]
```

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
from owslib.csw import CatalogueServiceWeb


endpoint = 'https://data.ioos.us/csw'

csw = CatalogueServiceWeb(endpoint, timeout=60)
```

The `csw` object created from `CatalogueServiceWeb` did not fetched anything yet.
It is the method `getrecords2` that uses the filter for the search. However, even though there is a `maxrecords` option, the search is always limited by the server side and there is the need to iterate over multiple calls of `getrecords2` to actually retrieve all records.
The `get_csw_records` does exactly that.

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
def get_csw_records(csw, filter_list, pagesize=10, maxrecords=1000):
    """Iterate `maxrecords`/`pagesize` times until the requested value in
    `maxrecords` is reached.
    """
    from owslib.fes import SortBy, SortProperty
    # Iterate over sorted results.
    sortby = SortBy([SortProperty('dc:title', 'ASC')])
    csw_records = {}
    startposition = 0
    nextrecord = getattr(csw, 'results', 1)
    while nextrecord != 0:
        csw.getrecords2(constraints=filter_list, startposition=startposition,
                        maxrecords=pagesize, sortby=sortby)
        csw_records.update(csw.records)
        if csw.results['nextrecord'] == 0:
            break
        startposition += pagesize + 1  # Last one is included.
        if startposition >= maxrecords:
            break
    csw.records.update(csw_records)
```

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
get_csw_records(csw, filter_list, pagesize=10, maxrecords=1000)

records = '\n'.join(csw.records.keys())
print('Found {} records.\n'.format(len(csw.records.keys())))
for key, value in list(csw.records.items()):
    print('[{}]\n{}\n'.format(value.title, key))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Found 39 records.
    
    [SREF CONUS 40km Ensemble Derived Products (Bias Corrected)/Best SREF CONUS 40km Ensemble Derived Products (Bias Corrected) Time Series]
    edu.ucar.unidata:grib/NCEP/SREF/CONUS_40km/ensprod_biasc/Best
    
    [UCSC California Current System ROMS Nowcast 10km]
    UCSC
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9432780 station, Charleston, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9432780
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9435380 station, South Beach, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9435380
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9437540 station, Garibaldi, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9437540
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9439040 station, Astoria, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9439040
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9440910 station, Toke Point, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9440910
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9441102 station, Westport, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9441102
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9442396 station, La Push, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9442396
    
    [FNMOC COAMPS Northeast Pacific/Best FNMOC COAMPS Northeast Pacific Time Series]
    edu.ucar.unidata:grib/FNMOC/COAMPS/Northeast_Pacific/Best
    
    [DGEX CONUS 12km/Best DGEX CONUS 12km Time Series]
    edu.ucar.unidata:grib/NCEP/DGEX/CONUS_12km/Best
    
    [Rapid Refresh CONUS 40km/Best Rapid Refresh CONUS 40km Time Series]
    edu.ucar.unidata:grib/NCEP/RAP/CONUS_40km/Best
    
    [NAM CONUS 12km from NOAAPORT/Best NAM CONUS 12km from NOAAPORT Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/CONUS_12km/Best
    
    [Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near SCRIPPS NEARSHORE, CA from 2015/01/07 23:00:00 to 2016/12/19 00:01:25.]
    edu.ucsd.cdip:CDIP_201p1_20150107-20161219
    
    [Rapid Refresh CONUS 20km/Best Rapid Refresh CONUS 20km Time Series]
    edu.ucar.unidata:grib/NCEP/RAP/CONUS_20km/Best
    
    [Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near ASTORIA CANYON, OR from 2016/03/30 17:00:00 to 2016/12/19 00:02:25.]
    edu.ucsd.cdip:CDIP_179p1_20160330-20161219
    
    [NAM CONUS 80km/Best NAM CONUS 80km Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/CONUS_80km/Best
    
    [Rapid Refresh CONUS 13km/Best Rapid Refresh CONUS 13km Time Series]
    edu.ucar.unidata:grib/NCEP/RAP/CONUS_13km/Best
    
    [Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near OCEAN STATION PAPA from 2015/01/01 01:00:00 to 2016/12/19 00:00:43.]
    edu.ucsd.cdip:CDIP_166p1_20150101-20161219
    
    [SREF Alaska 45km Ensemble Derived Products/Best SREF Alaska 45km Ensemble Derived Products Time Series]
    edu.ucar.unidata:grib/NCEP/SREF/Alaska_45km/ensprod/Best
    
    [Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near GRAYS HARBOR, WA from 2016/03/16 22:00:00 to 2016/12/18 23:53:56.]
    edu.ucsd.cdip:CDIP_036p1_20160316-20161218
    
    [NAM Alaska 45km from CONDUIT/Best NAM Alaska 45km from CONDUIT Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/Alaska_45km/conduit/Best
    
    [NAM CONUS 40km/Best NAM CONUS 40km Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/CONUS_40km/conduit/Best
    
    [GFS CONUS 80km/Best GFS CONUS 80km Time Series]
    edu.ucar.unidata:grib/NCEP/GFS/CONUS_80km/Best
    
    [NAM Alaska 22km/Best NAM Alaska 22km Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/Alaska_22km/Best
    
    [NCEP HRRR CONUS 2.5km/Best NCEP HRRR CONUS 2.5km Time Series]
    edu.ucar.unidata:grib/NCEP/HRRR/CONUS_2p5km/Best
    
    [Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near CLATSOP SPIT, OR from 2016/10/12 17:00:00 to 2016/12/19 00:04:07.]
    edu.ucsd.cdip:CDIP_162p1_20161012-20161219
    
    [COAMPS Ground and Sea Surface Temperature, 4km]
    COAMPS_4KM_GRND_SEA_TEMP
    
    [Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near UMPQUA OFFSHORE, OR from 2016/12/02 21:00:00 to 2016/12/18 23:49:37.]
    edu.ucsd.cdip:CDIP_139p1_20161202-20161218
    
    [HYbrid Coordinate Ocean Model (HYCOM): Global]
    hycom_global
    
    [NAM Alaska 95km/Best NAM Alaska 95km Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/Alaska_95km/Best
    
    [NAM Alaska 11km/Best NAM Alaska 11km Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/Alaska_11km/Best
    
    [COAMPS Visibility Forecast, 4km]
    COAMPS_4KM_VISIB
    
    [NOAA/NCEP Global Forecast System (GFS) Atmospheric Model]
    ncep_global
    
    [FNMOC COAMPS Southern California/Best FNMOC COAMPS Southern California Time Series]
    edu.ucar.unidata:grib/FNMOC/COAMPS/Southern_California/Best
    
    [NAM Alaska 45km from NOAAPORT/Best NAM Alaska 45km from NOAAPORT Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/Alaska_45km/noaaport/Best
    
    [G1SST, 1km blended SST]
    G1_SST_GLOBAL
    
    [GFS CONUS 20km/Best GFS CONUS 20km Time Series]
    edu.ucar.unidata:grib/NCEP/GFS/CONUS_20km/Best
    
    [NAM CONUS 20km/Best NAM CONUS 20km Time Series]
    edu.ucar.unidata:grib/NCEP/NAM/CONUS_20km/noaaport/Best
    

</pre>
</div>
That search returned a lot of records!
What if the user is not interested in those model results nor global dataset?
We can those be excluded  from the search with a `fes.Not` filter.

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
kw = dict(
    wildCard='*',
    escapeChar='\\\\',
    singleChar='?',
    propertyname='apiso:AnyText')


filter_list = [
    fes.And(
        [
            bbox_crs,  # Bounding box
            begin, end,  # start and end date
            or_filt,  # or conditions (CF variable names).
            fes.Not([fes.PropertyIsLike(literal='*NAM*', **kw)]),  # no NAM results
            fes.Not([fes.PropertyIsLike(literal='*CONUS*', **kw)]),  # no NAM results
            fes.Not([fes.PropertyIsLike(literal='*GLOBAL*', **kw)]),  # no NAM results
            fes.Not([fes.PropertyIsLike(literal='*ROMS*', **kw)]),  # no NAM results
        ]
    )
]

get_csw_records(csw, filter_list, pagesize=10, maxrecords=1000)

records = '\n'.join(csw.records.keys())
print('Found {} records.\n'.format(len(csw.records.keys())))
for key, value in list(csw.records.items()):
    print('[{}]\n{}\n'.format(value.title, key))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Found 12 records.
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9441102 station, Westport, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9441102
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9442396 station, La Push, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9442396
    
    [FNMOC COAMPS Northeast Pacific/Best FNMOC COAMPS Northeast Pacific Time Series]
    edu.ucar.unidata:grib/FNMOC/COAMPS/Northeast_Pacific/Best
    
    [COAMPS Ground and Sea Surface Temperature, 4km]
    COAMPS_4KM_GRND_SEA_TEMP
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9435380 station, South Beach, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9435380
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9439040 station, Astoria, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9439040
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9432780 station, Charleston, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9432780
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9437540 station, Garibaldi, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9437540
    
    [COAMPS Visibility Forecast, 4km]
    COAMPS_4KM_VISIB
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9440910 station, Toke Point, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9440910
    
    [FNMOC COAMPS Southern California/Best FNMOC COAMPS Southern California Time Series]
    edu.ucar.unidata:grib/FNMOC/COAMPS/Southern_California/Best
    
    [SREF Alaska 45km Ensemble Derived Products/Best SREF Alaska 45km Ensemble Derived Products Time Series]
    edu.ucar.unidata:grib/NCEP/SREF/Alaska_45km/ensprod/Best
    

</pre>
</div>
12 Records. That's better. But if the user is interested in only some specific service, it is better to filter by a string, like [`CO-OPS`](https://tidesandcurrents.noaa.gov/).

<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
filter_list = [
    fes.And(
        [
            bbox_crs,  # Bounding box
            begin, end,  # start and end date
            or_filt,  # or conditions (CF variable names).
            fes.PropertyIsLike(literal='*CO-OPS*', **kw),  # must have CO-OPS
        ]
    )
]

get_csw_records(csw, filter_list, pagesize=10, maxrecords=1000)

records = '\n'.join(csw.records.keys())
print('Found {} records.\n'.format(len(csw.records.keys())))
for key, value in list(csw.records.items()):
    print('[{}]\n{}\n'.format(value.title, key))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Found 7 records.
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9432780 station, Charleston, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9432780
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9435380 station, South Beach, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9435380
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9437540 station, Garibaldi, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9437540
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9439040 station, Astoria, OR]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9439040
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9440910 station, Toke Point, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9440910
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9441102 station, Westport, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9441102
    
    [urn:ioos:station:NOAA.NOS.CO-OPS:9442396 station, La Push, WA]
    opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9442396
    

</pre>
</div>
The easiest way to get more information is to explorer the individual records.
Here is the `abstract` and `subjects` from the station in Astoria, OR.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
import textwrap


value = csw.records['opendap.co-ops.nos.noaa.gov-urn_ioos_station_NOAA.NOS.CO-OPS_9439040']

print('\n'.join(textwrap.wrap(value.abstract)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    NOAA.NOS.CO-OPS Sensor Observation Service (SOS) Server  This station
    provides the following variables: Air pressure, Air temperature, Sea
    surface height amplitude due to equilibrium ocean tide, Sea water
    temperature, Water surface height above reference datum, Wind from
    direction, Wind speed, Wind speed of gust

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
print('\n'.join(value.subjects))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Air Temperature
    Barometric Pressure
    Conductivity
    Currents
    Datum
    Harmonic Constituents
    Rain Fall
    Relative Humidity
    Salinity
    Visibility
    Water Level
    Water Level Predictions
    Water Temperature
    Winds
    air_pressure
    air_temperature
    sea_surface_height_amplitude_due_to_equilibrium_ocean_tide
    sea_water_temperature
    water_surface_height_above_reference_datum
    wind_from_direction
    wind_speed
    wind_speed_of_gust
    climatologyMeteorologyAtmosphere

</pre>
</div>
The next step is to inspect the type services/schemes available for downloading the data. The easiest way to accomplish that is with by "sniffing" the URLs with `geolinks`.

<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
from geolinks import sniff_link

msg = 'geolink: {geolink}\nscheme: {scheme}\nURL: {url}\n'.format
for ref in value.references:
    print(msg(geolink=sniff_link(ref['url']), **ref))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    geolink: OGC:SOS
    scheme: Astoria
    URL: https://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?procedure=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&version=1.0.0&request=DescribeSensor&outputFormat=text/xml; subtype="sensorML/1.0.1/profiles/ioos_sos/1.0"&service=SOS
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/air_pressure&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/air_pressure&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/air_temperature&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/air_temperature&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/sea_surface_height_amplitude_due_to_equilibrium_ocean_tide&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/sea_surface_height_amplitude_due_to_equilibrium_ocean_tide&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/sea_water_temperature&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/sea_water_temperature&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/water_surface_height_above_reference_datum&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/water_surface_height_above_reference_datum&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/wind_from_direction&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/wind_from_direction&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/wind_speed&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/wind_speed&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/wind_speed_of_gust&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: OGC:SOS
    scheme: WWW:LINK - text/xml
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/xml;subtype="om/1.0.0/profiles/ioos_sos/1.0"&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/wind_speed_of_gust&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    
    geolink: None
    scheme: WWW:LINK
    URL: https://tidesandcurrents.noaa.gov/images/stationphotos/9439040A.jpg
    
    geolink: None
    scheme: WWW:LINK
    URL: https://tidesandcurrents.noaa.gov/publications/NOAA_Technical_Report_NOS_CO-OPS_030_QC_requirements_doc(revised)-11102004.pdf
    
    geolink: None
    scheme: WWW:LINK
    URL: https://tidesandcurrents.noaa.gov/stationhome.html?id=9439040
    
    geolink: OGC:SOS
    scheme: OGC:SOS
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?request=GetCapabilities&acceptVersions=1.0.0&service=SOS
    

</pre>
</div>
There are many direct links to Comma Separated Value (`CSV`) and
eXtensible Markup Language (`XML`) responses to the various variables available in that station. 

In addition to those links, there are three very interesting links for more information: 1.) the QC document, 2.) the station photo, 3.) the station home page.


For a detailed description of what those `geolink` results mean check the [lookup](https://github.com/OSGeo/Cat-Interop/blob/master/LinkPropertyLookupTable.csv) table.


![](https://tidesandcurrents.noaa.gov/images/stationphotos/9439040A.jpg)

The original search was focused on sea water temperature,
so there is the need to extract only the endpoint for that variable.

PS: see also the [pyoos example](http://ioos.github.io/notebooks_demos/notebooks/2016-10-12-fetching_data/) for fetching data from `CO-OPS`.

<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
start, stop
```




    (datetime.datetime(2016, 12, 12, 20, 49, 29, 121653),
     datetime.datetime(2016, 12, 19, 20, 49, 29, 121653))



<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
for ref in value.references:
    url = ref['url']
    if 'csv' in url and 'sea' in url and 'temperature' in url:
        print(msg(geolink=sniff_link(url), **ref))
        break
```
<div class="output_area"><div class="prompt"></div>
<pre>
    geolink: OGC:SOS
    scheme: WWW:LINK - text/csv
    URL: http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&service=SOS&responseFormat=text/csv&version=1.0.0&request=GetObservation&observedProperty=http://mmisw.org/ont/cf/parameter/sea_water_temperature&eventTime=2016-12-18T02:58:01/2016-12-18T04:58:01
    

</pre>
</div>
Note that the URL returned by the service has some hard-coded start/stop dates.
It is easy to overwrite those with the same dates from the filter.

<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
fmt = ('http://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/SOS?'
       'service=SOS&'
       'eventTime={0:%Y-%m-%dT00:00:00}/{1:%Y-%m-%dT00:00:00}&'
       'observedProperty=http://mmisw.org/ont/cf/parameter/sea_water_temperature&'
       'version=1.0.0&'
       'request=GetObservation&offering=urn:ioos:station:NOAA.NOS.CO-OPS:9439040&'
       'responseFormat=text/csv')

url = fmt.format(start, stop)
```

Finally, it is possible to download the data directly into a data `pandas` data frame and plot it.

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
import io
import requests
import pandas as pd

r = requests.get(url)

df = pd.read_csv(io.StringIO(r.content.decode('utf-8')),
                 index_col='date_time', parse_dates=True)
```

<div class="prompt input_prompt">
In&nbsp;[16]:
</div>

```python
%matplotlib inline
import matplotlib.pyplot as plt


fig, ax = plt.subplots(figsize=(11, 2.75))
ax = df['sea_water_temperature (C)'].plot(ax=ax)
ax.set_xlabel('')
ax.set_ylabel(r'Sea water temperature ($^\circ$C)')
ax.set_title(value.title)
```




    <matplotlib.text.Text at 0x7fdc6803ac50>




![png](data:image/png;base64,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
YII=
)


<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2016-12-19-exploring_csw.ipynb)
this notebook, or see a static view [here](http://nbviewer.ipython.org/urls/raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2016-12-19-exploring_csw.ipynb).
