from dash import Dash, dcc, html, Input, Output
import plotly.graph_objects as go
import pandas as pd
from plotly.subplots import make_subplots
import plotly.graph_objects as go
import io
from datetime import datetime

app = Dash(__name__)
  
df = pd.read_csv('Bunds.csv')

#converting date into yyyy-mm-dd
df['Date'] = pd.to_datetime(df['Date'])

df['Previous Close'] = df['Close'].shift()

# The 3 calculations of True Range 
df['High - Low'] = df['High'] - df['Low']
df['High - PClose'] = abs(df['High'] - df['Previous Close'])
df['Low - PClose'] = abs(df['Low'] - df['Previous Close'])

# Compute the max out of the 3
df['True_Range'] = df[['High - Low','High - PClose','Low - PClose']].max(axis=1)

# get the average TR for the previous 14 days or days_concerned 
df['ATR'] = df.True_Range.rolling(window = 14).mean() 

#Calculating std deviation for closing price
df['sdc']= df['Close'].rolling(window=14).std()

#Calculating std deviation for ATR
df['sdATR']= df['ATR'].rolling(window=20).std()


app.layout = html.Div([
     html.Div([
        html.H2("OHLC chart with technical indicators"),
        html.Img(src="/assets/stock-icon.png")
    ], className="banner"),
    
    dcc.DatePickerRange(
                    id="date-range",
                    min_date_allowed=df.Date.min().date(),
                    max_date_allowed=df.Date.max().date(),
                    start_date=df.Date.min().date(),
                    end_date=df.Date.max().date(),
                ),
    html.Br(),
    dcc.Graph(id="graph"),
    html.Br(),
    html.Div([
    html.H3("Done by Aanand Thakur"),
    html.H3("Email- anandthakur142@gmail.com")],
    className="bannerH3")
    
])

app.css.append_css({
    "external_url":"/assets/stylesheet.css"
})


@app.callback(
    Output('graph', 'figure'),
    [
      Input('date-range', 'start_date'),
      Input('date-range', 'end_date'),
    ],
)

def display_candlestick(start_date,end_date):
  mask = (
    (df.Date >= start_date) & (df.Date <= end_date)
  )
  dff = df.loc[mask, :]  
  
  fig = go.Figure()

#Plot1

  fig = make_subplots(rows=4, cols=1, shared_xaxes=True, vertical_spacing=0.1, row_heights=[0.7,0.4,0.4, 0.4], subplot_titles=("Candlestick with overlaid ATR","Avg True Range", "Moving std dev for close price", "Moving std dev for ATR"))
  
  fig.append_trace(go.Candlestick(x=dff['Date'],
                    open=dff['Open'],
                    high=dff['High'] ,
                    low=dff['Low'],
                    close=dff['Close'],
                    name="Candlestick")
                    , row=1 , col=1 )  

  fig.append_trace(
    go.Scatter(
        x=dff['Date'],
        y=dff['ATR'],
        name="Overlaid ATR",
        opacity=0.7, 
        line=dict(color='Green', width=2),
        ), 1,1 )
#Plot2
  fig.append_trace(
    go.Scatter(
        x=dff['Date'],
        y=dff['ATR'],
        name="ATR",
        opacity=0.7, 
        line=dict(color='Orange', width=2),
        ), 2,1 )

#Plot3

  fig.append_trace(
    go.Scatter(
        x=dff['Date'],
        y=dff['sdc'],
        name="Std dev for close price",
        opacity=0.7, 
        line=dict(color='Purple', width=2),
         ) , 3,1)

#Plot4

  fig.append_trace(
    go.Scatter(
        x=dff['Date'],
        y=dff['sdATR'],
        name="Std dev for ATR",
        opacity=0.7, 
        line=dict(color='Blue', width=2),
         ) , 4,1)

  fig.update_layout( 
         xaxis_rangeslider_visible=False,
         height=600, width=1200,
         template='plotly_dark',
         title="FGBL: 01/01/2018- 05/31/2022",
         yaxis_title="Price",
         legend_title="Indicators",
         font=dict(
        family="Courier New, monospace",
        size=16,
  
    )
           )

  return fig


app.run_server(host='0.0.0.0', port=8050, debug=False)
