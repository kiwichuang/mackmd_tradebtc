---
title: 'Project documentation template'
disqus: hackmd
---
加密貨幣交易策略
===
![downloads](https://img.shields.io/github/downloads/atom/atom/total.svg)
![build](https://img.shields.io/appveyor/ci/:user/:repo.svg)
![chat](https://img.shields.io/discord/:serverId.svg)

## Table of Contents

[TOC]

## Beginners Guide

If you are a total beginner to this, start here!

1. Visit crypt arsenal
2. Click "Sign in"
3. write a program to start


Project
---

```gherkin=
class Strategy():
    # option setting needed
    def __setitem__(self, key, value):
        self.options[key] = value

    # option setting needed
    def __getitem__(self, key):
        return self.options.get(key, '')

    def __init__(self):
        # strategy property
        self.subscribedBooks = {
            'Binance': {
                'pairs': ['BTC-USDT'],
            },
        }
        
        self.period = 1 *60 * 60
        self.options = {}
        
        #self.period=60*60


        # user defined class attribute
        self.last_type = 'sell'
        self.last_cross_status = False
        self.close_price_trace = np.array([])
        self.ma_long = 14
        self.ma_short = 5
        self.UP = True
        self.DOWN = False
        self.mult=1
        self.trigger=False
        self.triggerwait=False
        self.new_time = np.array([])
        
    def on_order_state_change(self,  order):
        Log("on order state change message: " + str(order) + " order price: " + str(order["price"]))

    
    def bband(self):
        up, mid, low = talib.BBANDS(self.close_price_trace, timeperiod=70, nbdevup=2 ,nbdevdn=2.1, matype=0)
        bband = ((self.close_price_trace- low) / (up - low))[-1]
        Log(str(bband))
        return float(bband)


    def stop(self):
        s_ema2 = talib.MA(self.close_price_trace,timeperiod=1,matype=1)[-2]
        s_ema = talib.MA(self.close_price_trace,timeperiod=1,matype=1)[-1]
        stop=(s_ema2*0.95)-s_ema
        
        return float(stop)


    def rsi(self):
        #close_price = information['candles'][exchange][pair][0]['close']
        rsi=talib.RSI(self.close_price_trace, 7)[-1]
        #Log(str(rsi))
        return float(rsi)

    def l_ema(self):

        l_ema = talib.MA(self.close_price_trace,timeperiod=9,matype=1)[-1]
        
        return float(l_ema)

    def s_ema(self):
      
        s_ema = talib.MA(self.close_price_trace,timeperiod=1,matype=1)[-1]

        return float(s_ema)




    def get_current_ma_cross(self):
        #s_ma = talib.SMA(self.close_price_trace, self.ma_short)[-1]
        s_ema = talib.MA(self.close_price_trace,timeperiod=1,matype=1)[-1]
        #s_ema=np.ndarray.tolist(s_ema)[-1]
        #l_ma = talib.SMA(self.close_price_trace, self.ma_long)[-1]
        l_ema = talib.MA(self.close_price_trace,timeperiod=9,matype=1)[-1]
        #l_ema=np.ndarray.tolist(l_ema)[-1]
        Log(str(s_ema))
        Log(str(l_ema))
        #if np.isnan(s_ema) or np.isnan(l_ema):
        #    return None
        if s_ema > l_ema:
            self.last_cross_status = self.UP
            return self.last_cross_status
        if s_ema < l_ema:
            self.last_cross_status = self.DOWN
            return self.last_cross_status

    # called every self.period
    def trade(self, information):
        exchange = list(information['candles'])[0]
        pair = list(information['candles'][exchange])[0]
        target_currency = pair.split('-')[0]  #BTC
        base_currency = pair.split('-')[1]  #USDT
        base_currency_amount = self['assets'][exchange][base_currency] 
        target_currency_amount = self['assets'][exchange][target_currency] 
        # add latest price into trace
        close_price = information['candles'][exchange][pair][0]['close']
        open_price = information['candles'][exchange][pair][0]['open']
        high_price = information['candles'][exchange][pair][0]['high']
        low_price = information['candles'][exchange][pair][0]['low']
        volume_price = information['candles'][exchange][pair][0]['volume']
        time=information['candles'][exchange][pair][0]['time']
        self.close_price_trace = np.append(self.close_price_trace, [float(close_price)])
        # only keep max length of ma_long count elements
        #self.close_price_trace = self.close_price_trace[-self.ma_long:]
        #Log(str(self.close_price_trace))
        #Log(str(close_price))
        rsi=self.rsi()
        bband=self.bband()
        #s_sma=self.sma()
        stop=self.stop()
        get_current_ma_cross=self.get_current_ma_cross()
        s_ema=self.s_ema()
        l_ema=self.l_ema()

        pair = list(information['candles'][exchange])[0] #BTC-USDT
        target_currency = pair.split('-')[0]  #BTC
        base_currency = pair.split('-')[1]  #USDT

        base_currency_amount = self['assets'][exchange][base_currency]
        target_currency_amount = self['assets'][exchange][target_currency]
        #Log(str(base_currency_amount))
        #Log(str(target_currency_amount))
        buynum=float(base_currency_amount*0.05)/close_price
        #####Log(str(buynum))
        sellnum=float(-target_currency_amount*0.1)
        #####Log(str(sellnum))
        time=np.datetime64(time)
        # cross up
        #########################################Log(str(time))################################
        if self.trigger==True and self.triggerwait==False:
            self.new_time=time + np.timedelta64(5, 'h')
            Log(str(self.new_time))
            self.triggerwait=True
        if self.new_time==time:
            self.trigger=False
            self.triggerwait=False
            Log(str(time))
            
#2021-04-22T14:20:00Z

        if bband<0 or stop>0 and self.trigger==False:
            self.last_type = 'sell'
            self.trigger=True
            #self.sleep(1800)
            return [
                {
                    'exchange': exchange,
                    'amount': -target_currency_amount,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                }
            ] 

        if l_ema > s_ema and rsi<20 and self.trigger==False:
            self.last_type = 'buy'
            self.mult=3
            return [
                {
                    'exchange': exchange,
                    'amount': buynum*self.mult,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                }
            ]

        if l_ema > s_ema and rsi<30 and self.trigger==False:
            self.last_type = 'buy'
            self.mult=1
            return [
                {
                    'exchange': exchange,
                    'amount': buynum*self.mult,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                }
            ]
   
           
            
        # cross down
        if l_ema < s_ema and rsi>75 and self.trigger==False:
            self.last_type = 'sell'
            self.mult=1
            return [
                {
                    'exchange': exchange,
                    'amount': sellnum*self.mult,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                }
            ]

        if l_ema < s_ema and rsi>85 and self.trigger==False:
            self.last_type = 'sell'
            self.mult=3
            return [
                {
                    'exchange': exchange,
                    'amount': sellnum*self.mult,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                }
            ]
            
        return []
```
> 策略發想 停損 遮罩 時間回測一小時  [name=kiwi]



> Read more for my github here: ........

User flows
---
```sequence
kiwi->crypt arsenal server: when you see bband<0 or next sma broke 5% sell all

Note right of kiwi: kiwi think it is a stoploss Strategy
Note right of kiwi: kiwi stop buy or sell for few hours
kiwi->crypt arsenal server: if rsi and MACD match buy and sell


```

> this program only fit for this platform

Project picture
---

![](https://i.imgur.com/Lh78S6O.png)

![](https://i.imgur.com/56Xjt1S.png)




## Appendix and FAQ

:::info
**Find this document incomplete?** Leave a comment!
:::

###### tags: `加密貨幣` `交易策略`
