
**Time of the day analysis:**


```python
import numpy as np
import pandas as pd
import csv
import datetime
import itertools
import geopy
import requests
import folium as fl
import seaborn as sns
import matplotlib.pyplot as plt
%matplotlib inline
```

Downloading four weeks of MTA turnstile data between April 15 - May 6 (since the gala happens in Summer 2018, the canvassing needs to happen between now and Summer.  Accordingly, we want to base our analysis on how the MTA traffic looked like in 2017 in April-May). We read the data directly from the MTA site, using Pandas read_table and concat methods and storing in a table called mta


```python
mta = pd.DataFrame()
for i in [170506,170429, 170422, 170415]:
    link = 'http://web.mta.info/developers/data/nyct/turnstile/turnstile_'+str(i)+'.txt'
    mta = pd.concat([mta, pd.read_table(link, sep=',')], ignore_index=True)
```

We define the below function to **clean up** the data.  Please refer to individual comments below on how each line of code cleans the dataframe.


```python
def clean_abnormals(mta):
    
    mta.sort_values(by=['C/A','UNIT','STATION','SCP','DATE','TIME'],inplace=True)
    
    #the EXITS column header has blank spaces which we will remove by using the rename method.
    mta.rename(columns={'EXITS                                                               ': 'EXITS'}, inplace=True)
    
    #the below is to create two new columns - one for entries per turnstile, one for exits per turnstile - to get the true count of entries and exits within the 4 hour window
    mta['ENTRIES-1']=mta.groupby(['C/A','UNIT','STATION','SCP'])['ENTRIES'].diff().fillna(0)
    mta['EXITS-1']=mta.groupby(['C/A','UNIT','STATION','SCP'])['EXITS'].diff().fillna(0)
    

    #the below two lines are to remove entries and exits corresponding to 'counter jumps' where the entry/ exit counter resets to a distant and unrelated sequence, resulting in abnormal entry and exit values.
    #the logic used is that in a four hour window, a maximum of 4 (hours)*60 (minutes)*60 (seconds) = 14,400 people can enter or exit through a turnstile (not more than one per second)
    mta.loc[abs(mta['ENTRIES-1'])>14400,'ENTRIES-1']=0
    mta.loc[abs(mta['EXITS-1'])>14400,'EXITS-1']=0

    #the below code will help convert negative entries and exits values into positive values. Negative values are a result of the counters going backwards sometimes.
    mta.loc[mta['ENTRIES-1']<0,'ENTRIES-1']=abs(mta['ENTRIES-1'])
    mta.loc[mta['EXITS-1']<0,'EXITS-1']=abs(mta['EXITS-1'])

    #we will then create a new column for date-time.  this column will be helpful for time-series analysis or to simply extract date/ day/ hour information
    mta['date-time'] = mta['DATE'] + ' ' + mta['TIME']
    mta['date-time']=pd.to_datetime(mta['date-time'])
    
    #the entry/ exit timestamps across stations are not standardized.  Some of the stations have the [0,4,8...20] sequence while other stations have other sequences.
    #the below lines will standardize the hours and convert them into 4-hour buckets
    mta['HOUR']=mta['date-time'].dt.hour
    mta['HOUR'] = pd.cut(mta['HOUR'], [0,4,8,12,16,20,24])
    mta['HOUR'] = mta['HOUR'].apply(lambda x:str(x))
    
    #the below line creates a new column for 'Net Entries' which is critical for the 'Time of the day' analysis which follows below.
    mta['ENTRIES-EXITS'] = mta['ENTRIES-1']-mta['EXITS-1']
    
    #the below line extracts the 'day of the week' info from the newly created date-time column.  'Day of the week' column helps segregate weekday commuter patterns from weekend patterns.
    mta['DAY'] = mta['date-time'].apply(lambda x: x.weekday())
    
    return mta
```


```python
mta = clean_abnormals(mta)
```

**Analysis for Weekdays:**


```python
top_stations = list(mta.groupby(by='STATION')['ENTRIES-1'].sum().sort_values(ascending=False)[:5].index)
plt.figure(figsize=(14,10))
sns.set_style('whitegrid')
for station in top_stations:
    td1 = (mta[(mta['STATION']==station)&(mta['DAY']<5)].groupby(by=['HOUR'])['ENTRIES-EXITS'].sum())/20
    td1.plot(label=station, lw=4)
    plt.axhline(linewidth=2, color='k', linestyle='--')
    plt.legend(loc='upper left',fontsize=15)
plt.xticks(np.arange(6), ('0-4am', '4-8am', '8am-12pm', '12-4pm', '4-8pm', '8pm-12am'), rotation=90, fontsize=10)
plt.xticks(fontsize=14)
plt.yticks(fontsize=14)
plt.xlabel('Time of Day', fontsize=18)
plt.ylabel('Average Daily Net Entries (Entries minus Exits)', fontsize=18)
plt.tight_layout();
```


![png](output_8_0.png)


**Analysis for Weekends:**


```python
list(mta.groupby(by='STATION')['ENTRIES-1'].sum().sort_values(ascending=False)[:6].index)+list(mta.groupby(by='STATION')['ENTRIES-1'].sum().sort_values(ascending=False)[7:11].index)
```




    ['34 ST-PENN STA',
     'GRD CNTRL-42 ST',
     '34 ST-HERALD SQ',
     '23 ST',
     '14 ST-UNION SQ',
     '42 ST-PORT AUTH',
     'FULTON ST',
     '86 ST',
     '125 ST',
     'CANAL ST']




```python
top_stations = list(mta.groupby(by='STATION')['ENTRIES-1'].sum().sort_values(ascending=False)[:5].index)
#+list(mta.groupby(by='STATION')['ENTRIES-1'].sum().sort_values(ascending=False)[7:11].index)
plt.figure(figsize=(14,10))
sns.set_style('darkgrid')
for station in top_stations:
    td1 = (mta[(mta['STATION']==station)&(mta['DAY']>4)].groupby(by=['HOUR'])['ENTRIES-EXITS'].sum())/8
    td1.plot(label=station, lw=4)
    plt.axhline(linewidth=2, color='k', linestyle='--')
    plt.legend(loc='lower right',fontsize=15)
plt.xticks(np.arange(6), ('0-4am', '4-8am', '8am-12pm', '12-4pm', '4-8pm', '8pm-12am'), rotation=90, fontsize=10)
plt.xticks(fontsize=14)
plt.yticks(fontsize=14)
plt.xlabel('Time of Day', fontsize=18)
plt.ylabel('Average Daily Net Entries (Entries minus Exits)', fontsize=18)
plt.tight_layout();
```


![png](output_11_0.png)

