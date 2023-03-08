# ExtractAPIandMakeDashBoard
import pandas as pd
from pandas_datareader import data as pdr
from datetime import date
import yfinance as yf
import streamlit as st
import plotly.graph_objects as go
import requests
import os
import plotly.express as px
yf.pdr_override()
# Name of tables from CSV file what we want to be showed at chart
balance_table = [
    'totalAssets',
    'totalLiabilities',
    'totalEquity'
]

cash_table = [
    'netIncome',
    'netCashUsedForInvestingActivites',
    'netCashUsedProvidedByFinancingActivities',
    'netChangeInCash'
]

# Function to extract DATA for our stocks


def get_data(ticker: str):

    try:
        # Using pandas to get data about our ticker
        price_data = pdr.get_data_yahoo(ticker)
        # Using API extraction with replaysing ticker name in link for getting DATA about balance and cash statments of the ticker we will write in dashboard
        respond2 = requests.get(
            f'https://financialmodelingprep.com/api/v3/balance-sheet-statement/{ticker}?apikey=76eb66a272dcca9d978d5b4f00a257fd&limit=120')
        balance = respond2.json()
        respond3 = requests.get(
            f'https://financialmodelingprep.com/api/v3/cash-flow-statement/{ticker}?apikey=76eb66a272dcca9d978d5b4f00a257fd&limit=120')
        cash = respond3.json()
        # Creating new CSV file with data we extracting in folder /data/
        price_data.to_csv(f'data/{ticker}_price.csv')
        # using pandas normalizer to make json_file easy extractable into CSV
        df2 = pd.json_normalize(balance)
        df3 = pd.json_normalize(cash)
        df2.to_csv(f'data/{ticker}_balance.csv', index=True)
        df3.to_csv(f'data/{ticker}_cash.csv', index=True)
    except Exception as e:
        st.write(e)
    return (
        price_data.reset_index()
    )

# Function for Main price chart vusuals with Moving average in it


def get_candlestick_chart(price_data, ma_type, ma_length, plot_days):

    if ma_type == 'Simple':
        price_data['ma'] = price_data['Close'].rolling(int(ma_length)).mean()
    else:
        price_data['ma'] = price_data['Close'].ewm(int(ma_length)).mean()

    price_data = price_data[-int(plot_days):]

    fig = go.Figure()
    # Asigning OUR colums from CSV file into the variables for ploting the chart
    fig.add_trace(
        go.Candlestick(
            x=price_data['Date'],
            open=price_data['Open'],
            high=price_data['High'],
            low=price_data['Low'],
            close=price_data['Close'],
            showlegend=False,
        )
    )
    # Assignning colums for moving average
    fig.add_trace(
        go.Line(
            x=price_data['Date'],
            y=price_data['ma'],
            name=f'{int(ma_length)} Period {ma_type} Moving Average'
        )
    )
    # Creat better visible chart with our days input not all data
    fig.update_xaxes(
        rangebreaks=[{'bounds': ['sat', 'mon']}],
        rangeslider_visible=False,
    )
    # Legent orientation and size of the chart
    fig.update_layout(
        legend={'x': 0, 'y': -0.05, 'orientation': 'h'},
        margin={'l': 50, 'r': 50, 'b': 50, 't': 25},
        width=800,
        height=800,
    )

    return fig

# Function for Balance Sheet chart


def plot_balance_sheet(df2):

    fig = go.Figure()
    # We inser our names for balance_sheet from our CSV
    for col in balance_table:

        fig.add_trace(
            go.Line(
                x=df2['calendarYear'],
                # Assign our colums from CSV to be in Y axis
                y=df2[col],
                # Name equal to the colum names
                name=col,
            )
        )
    # Add this to make it look like a trend
    fig.update_yaxes(type='log')
    # Change our names from colum to a new one, better readable because they will be seeing in the legent
    newname = {'totalAssets': 'Total Assets',
               'totalLiabilities': 'Total Liabilities', 'totalEquity': 'Total Equity'}
    fig.for_each_trace(lambda t: t.update(name=newname[t.name],
                                          legendgroup=newname[t.name]
                                          )
                       )

    fig.update_layout(
        legend={'x': 0, 'y': -0.10, 'orientation': 'h'},
        margin={'l': 50, 'r': 50, 'b': 50, 't': 25},
        width=800,
        height=400,
    )

    return fig

# Function for Cash sheet chart


def plot_cash_sheet(df3):

    fig = go.Figure()

    for col in cash_table:

        fig.add_trace(
            go.Line(
                x=df3['calendarYear'],
                y=df3[col],
                name=col,
            )
        )
    newname = {'netIncome': 'Net Income', 'netCashUsedForInvestingActivites': 'Cash For Investing Activities',
               'netCashUsedProvidedByFinancingActivities': 'Cash For Financing Activities', 'netChangeInCash': 'Net Change In Cash'}
    fig.for_each_trace(lambda t: t.update(name=newname[t.name],
                                          legendgroup=newname[t.name]
                                          )
                       )

    fig.update_layout(
        legend={'x': 0, 'y': -0.10, 'orientation': 'h'},
        margin={'l': 50, 'r': 50, 'b': 50, 't': 25},
        width=800,
        height=400,
    )

    return fig


# Sidebar controls -----------------------------------------------------------
# Creat an input for our Ticker wich we will use in our functions
ticker = st.sidebar.text_input(
    label='Stock ticker',
    value='TSLA',
)
# Creat a box for selection Type of the Moving Average
ma_type = st.sidebar.selectbox(
    label='Moving average type',
    options=['Simple', 'Exponential'],
)
# Creat an input for Moving average (how many days)
ma_length = st.sidebar.number_input(
    label='Moving average length',
    value=10,
    min_value=2,
    step=1,
)
# input for days on chart to show on a graf
plot_days = st.sidebar.number_input(
    label='Chart viewing length',
    value=120,
    min_value=1,
    step=1,
)
# Add to update the data
st.sidebar.button(
    label='UpdateData',
    on_click=get_data,
    kwargs={'ticker': ticker}
)

# The dashboard plots --------------------------------------------------------

st.header(f'{ticker} Dashboard')

# Check if we have the stock data, if not, download it
# For Ticker price
if os.path.isfile(f'data/{ticker}_price.csv'):
    price_data = pd.read_csv(f'data/{ticker}_price.csv')
else:
    price_data = get_data(ticker)
# For Balance sheet
if os.path.isfile(f'data/{ticker}_balance.csv'):
    df2 = pd.read_csv(f'data/{ticker}_balance.csv')
else:
    df2 = get_data(ticker)
# For Cash sheet
if os.path.isfile(f'data/{ticker}_cash.csv'):
    df3 = pd.read_csv(f'data/{ticker}_cash.csv')
else:
    df3 = get_data(ticker)

st.subheader('Price chart')
# inserting our function in to ploty_chart
st.plotly_chart(
    get_candlestick_chart(price_data, ma_type, ma_length, plot_days)
)
st.subheader('Balance Sheet Chart ')
st.plotly_chart(
    plot_balance_sheet(df2)
)
st.subheader('Cash Sheet Chart ')
st.plotly_chart(
    plot_cash_sheet(df3)
)
