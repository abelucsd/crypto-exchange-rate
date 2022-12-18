## Crypto Comparison
The goal of this notebook is to show an analysis which incites a prompt to the user.
Due to our technological advances, we have a desire to gather data and use them as information. However, there is an art in how we present data to our clients. Our data must be able to show and tell a story to the audience.


```python
import json, hmac, hashlib, time, requests, json, datetime
import pandas as pd
from datetime import date
from IPython.display import JSON
import seaborn as sbs
```


```python
# API Service Name: "coingecko"
# Get API request
def api_coins_list():
    '''
        Store crypto ids into a text file due to limited API calls.
    '''
    api_url = 'https://api.coingecko.com/api/v3/coins/list?include_platform=true'
    all_data = []
    r = requests.get(api_url)   
    with open("coin_ids.txt", "w") as f:
        json.dump(r.json(), f)

```


```python
def load_file():
    f = open("coin_ids.txt", "r", encoding="utf8")    
    return f.read()
```


```python
JSON(json.loads(load_file()))
```




    <IPython.core.display.JSON object>




```python
def get_data():
    '''
        Convert to API return into csv
    '''
    f = open("coin_ids.txt", "r", encoding="utf8")        
    json_data = f.read()
    currencies = json.loads(json_data)
    all_data = []
    # id, symbol, name, platform
    for item in currencies:
        data = {
            'id': item['id'],
            'name': item['name']
        }
        all_data.append(data)
    df = pd.DataFrame(all_data)
    df.to_csv('coin_data.txt', header=True, index=None, mode='w')
    
```


```python
get_data()
```

## Prompts

##### Compared to a certain coin, should the user trade user's current coin by looking at the historical trend?


```python
def get_coin_market_chart(coin_id):
    """
        Params: str - coin_id
        Return: str - coin name, list [str, str] - coin market chart [timestamp in ms, price]
    """
    coin_ids_df = pd.read_csv("coin_data.txt")
        
    if not coin_ids_df['id'].eq(coin_id).any():        
        return [None, None]
    coin_name = coin_ids_df.loc[coin_ids_df['id'] == coin_id].values[0][1]
        
    # timestamp at 12 AM
    today = date.today()
    timestamp = time.mktime(today.timetuple())
    
    # market in the past 7 days
    day_in_seconds = 86400     
    timestamp -= day_in_seconds * 7
    
    request_url = 'https://api.coingecko.com/api/v3/coins/{coin}/market_chart?vs_currency=usd&days=14&interval=daily'.format(coin=coin_id)
    # the last index is the current time. [:-2] 12 AM of current date.
    r = requests.get(request_url)        
    
    return [coin_name, r.json()["prices"]]
```


```python
def week_timestamps():
    """
        Return: list 
                    - 0, 1 this week's mon to current day timestamp
                    - 2, 3 last week time stamps mon to fri
    """
    from datetime import date, datetime, timedelta

    today = date.today()                
    day_in_seconds = 86400     
    timestamp_today = time.mktime(today.timetuple())            
    timestamp_this_start = timestamp_today - day_in_seconds * (today.weekday() + 1)
    if today.weekday() > 5:        
        timestamp_this_end = timestamp_this_start + day_in_seconds * 5
    else:
        timestamp_this_end = time.mktime(today.timetuple())
    
    # last week
    timestamp_last_start = timestamp_this_start - day_in_seconds * 7
    timestamp_last_end = timestamp_this_start - day_in_seconds * 2
    
    return [int(timestamp_this_start), int(timestamp_this_end),\
           int(timestamp_last_start), int(timestamp_last_end)]
```


```python
week_timestamps()
```




    [1670734800, 1671166800, 1670130000, 1670562000]




```python
def day_choice(day):
    match day:
        case 0:
            return 'Mon'
        case 1:
            return 'Tue'
        case 2:
            return 'Wed'
        case 3:
            return 'Thu'
        case 4:
            return 'Fri'
        case _:
            return 'None'                

def get_prices(crypto_id):
    '''
        Params: str - crypto id
        Return: str - bitcoin name, dataframe - this week market , dataframe - last week market
    '''
    
    
    coin_chart = get_coin_market_chart(crypto_id)
    timestamps = week_timestamps()   
    if coin_chart[0] is None:
        return None
            
    # get items    
    last_week_data, this_week_data = [], []    
    day_counter_last, day_counter_this = 0, 0
    for item in coin_chart[1]:
        timestamp = int(item[0] // 1000)
        # print("%s < %s < %s" % (timestamps[0], timestamp, timestamps[1]))
        
        if timestamps[0] <= timestamp <= timestamps[1]:            
            data = {
                'date': datetime.date.fromtimestamp(timestamp),
                'day': day_choice(day_counter_this),
                'price': item[1],
                'week': 'this'
            }
            # print("This Timestamp: %s | day: %s" % (timestamp, day_counter_this))
            day_counter_this += 1
            this_week_data.append(data)            
        elif timestamps[2] <= timestamp <= timestamps[3]:               
            data = {
                'date': datetime.date.fromtimestamp(timestamp),
                'day': day_choice(day_counter_last),
                'price': item[1],
                'week': 'last'
            }
            # print("Last Timestamp: %s | day: %s" % (timestamp, day_counter_last))
            day_counter_last += 1
            last_week_data.append(data)

    last_week_df = pd.DataFrame(last_week_data)
    this_week_df = pd.DataFrame(this_week_data)
    
    return [coin_chart[0], this_week_df, last_week_df]    
```

### Data Analysis
The graph below shows this week's coin data versus last week's.
The coloring of the graph lines should pop to the user and quickly give away whether the current week's price is higher or lower than last week's.
In a quick glance of the graph, questions should arise from the user:
* Bitcoin is higher this week, perhaps I am doing well and I should keep.
* Bitcoin is surging, perhaps this is a trend. Should I buy more coins?
* Bitcoin lowered from last week; however, it is not that much lower. Perhaps, I should keep.
* Bitcoin is losing a lot in price. I should definitely sell.



```python
import matplotlib.pylab as plt
def plot_crypto_data(crypto_id):
    market_prices = get_prices(crypto_id)
    if market_prices[0] is None:
        print("Invalid coin id")
        return
    coin_name, this_week_df, last_week_df = market_prices[0], market_prices[1], market_prices[2]    
    #plt = sbs.barplot(x = 'date', y = 'price', data = last_week_df)
    combined_df = pd.concat([this_week_df, last_week_df], ignore_index=True)
    plot = sbs.relplot(
        data=combined_df, kind="line", 
        hue='week', palette=['blue', 'lightgrey'],
        x="day", y="price", legend='auto',         
    ).set(title='%s Price: Current vs Last Week' % (coin_name))    
    
    plt.xticks(rotation=45)
```


```python
plot_crypto_data('bitcoin')
```


    
![png](currency-exchange-Copy1_files/currency-exchange-Copy1_15_0.png)
    


References: 
Nussbaumer Knaflic, Cole (2015). Storytelling with Data
    
