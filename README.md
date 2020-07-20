# InsightMap
The results will be the quantitative values along with maps depicting relevant geographies of consumer complaints.


import pandas as pd
import numpy as np
from IPython.display import display
import sqlite3
from sqlalchemy import create_engine
import string
from statsmodels.stats.multicomp import pairwise_tukeyhsd
from scipy import stats
from plotly.offline import iplot
import chart_studio.plotly as py
from plotly.tools import FigureFactory as FF
from plotly.graph_objs import Bar, Scatter, Marker, Layout, Choropleth, Histogram
display(pd.read_csv('complaints2021.csv', nrows=5).head())
db_conn = create_engine('sqlite:///complaints2021.db')
db_conn.connect()
print(db_conn)


chunks = 25000
print("=====")
df = pd.read_csv('complaints2021.csv', chunksize=chunks,
                        iterator=True, encoding='utf-8')
print("jjjjjj")

for data in df:
    print(data.columns)
    data = data.rename(columns={col: col.replace('-', ' ') for col in data.columns})
    data = data.rename(columns={col: col.strip() for col in data.columns})
    data = data.rename(columns={col: string.capwords(col) for col in data.columns})
    data = data.rename(columns={col: col.replace(' ', '') for col in data.columns})
    data.to_sql('data', db_conn,  if_exists='append')
print("+++++")




display(pd.read_sql_query('SELECT * FROM data LIMIT 10', db_conn))
query = pd.read_sql_query('SELECT Product, Company, COUNT(*) as `Complaints`'
                          'FROM data '
                          'GROUP BY Product ' 
                          'ORDER BY `Complaints` DESC', db_conn)


print("py.iplot")

iplot([Bar(x=query.Product, y=query.Complaints)], filename = 'ConsumerComplaints_Products with most complaints')



query_responses = pd.read_sql_query('SELECT Company, COUNT(*) as `Complaints` '
                           'FROM data '
                           'GROUP BY Company '
                           'ORDER BY `Complaints` DESC '
                           'LIMIT 10 ', db_conn)

iplot([Bar(x=query_responses.Company, y=query_responses.Complaints)], filename = 'ConsumerComplaints_Companies with most Complaints')

query_comp = pd.read_sql_query('SELECT Company, '
                           'COUNT(CASE WHEN `TimelyResponse?` = "Yes" THEN 1 ELSE NULL END) As YesCount, '
                           'COUNT(CASE WHEN `TimelyResponse?` = "No" THEN 1 ELSE NULL END) As NoCount, '
                           'COUNT(*) as Total '
                           'FROM data '
                           'GROUP BY Company '
                           'HAVING COUNT(*) > 500 '
                           'ORDER BY YesCount DESC', db_conn)
query_comp['Timely_Response_Ratio'] = query_comp.YesCount / query_comp.Total * 100
bot_10_response = query_comp.sort_values('Timely_Response_Ratio', ascending=True)[0:10]
responses = 'Timely Responses: ' + bot_10_response.YesCount.astype(str) + '<br>' + 'Untimely Responses: ' + bot_10_response.NoCount.astype(str)
iplot([Bar(x=bot_10_response.Company, y=bot_10_response.Timely_Response_Ratio,
              text=responses)], filename='ConsumerComplaints_LowestTimelyResponse')

query3 = pd.read_sql_query('SELECT Product, Issue, COUNT(*) `Number of Complaints` '
                          'FROM data '
                          'WHERE Product = "Mortgage" '
                          'GROUP BY Issue '
                          'ORDER BY `Number of Complaints` DESC', db_conn)
iplot([Bar(x=query3.Issue, y=query3['Number of Complaints'])], filename='ConsumerComplaints_MostComplainedProductIssue')

state_query = pd.read_sql_query('SELECT State, Product, COUNT(*) as `Complaints` '
                                'FROM data '
                                'WHERE State <> "None" '
                                'GROUP BY State', db_conn
                               )
dat = [dict(
    type = 'choropleth',
    locations = state_query['State'],
    z = state_query['Complaints'],
    locationmode = 'USA-states'
    )]

layout = dict(geo = dict(
    scope='usa', showlakes=True, lakecolor = 'rgb(255, 255, 255)'))

fig = dict(data=dat, layout=layout)

iplot(fig, filename='ConsumerComplaints_StatesWithMostComplaints')

query_comps = pd.read_sql_query('SELECT State, Company, COUNT(*) as `Complaints` '
                               'FROM data '
                               'WHERE State <> "None" '
                               'AND Company <> "None" '
                               'GROUP BY Company, State '
                               'ORDER BY `Complaints` DESC, Company, State', db_conn)

state_grouped = query_comps.groupby('State', as_index=False).first().sort_values('Complaints', ascending=False)
state_grouped['Company_Code'] = state_grouped['Company'].astype('category').cat.codes
company_codes = pd.DataFrame(state_grouped['Company'].unique(), index=state_grouped['Company_Code'].unique(), columns=['Code'])
dat = [dict(
    type = 'choropleth',
    locations = state_grouped['State'],
    z =  state_grouped['Company_Code'],
    locationmode = 'USA-states',
    text = state_grouped['Company'],
    colorscale = [[0.0, 'rgb(255,255,179)'], [0.1, 'rgb(141,211,199)'], [0.2, 'rgb(190,186,218)'],[0.3, 'rgb(251,128,114)'],\
            [0.4, 'rgb(128,177,211)'], [0.5, 'rgb(253,180,98)'], [0.6, 'rgb(179,222,105)'], [0.7, 'rgb(252,205,229)'],\
            [0.8, 'rgb(217,217,217)'], [0.9, 'rgb(188,128,189)'], [1, 'rgb(204,235,197)']],
    colorbar = dict(
        tickmode = 'array',
        tickvals = list(company_codes.index),
        ticktext = list(company_codes.Code))
    )]

layout = dict(
    geo = dict(
        scope='usa', showlakes=True, lakecolor = 'rgb(255, 255, 255)'))

fig = dict(data=dat, layout=layout)

iplot(fig, filename='ConsumerComplaints_MostComplainedCompanyByState')

query_products = pd.read_sql_query('SELECT State, Product, COUNT(*) as `Complaints` '
                               'FROM data '
                               'WHERE State <> "None" '
                               'AND Product <> "None" '
                               'GROUP BY Product, State '
                               'ORDER BY `Complaints` DESC, Product, State', db_conn)

product_grouped = query_products.groupby('State', as_index=False).first().sort_values('Complaints', ascending=False)
product_grouped['Product_Code'] = product_grouped['Product'].astype('category').cat.codes
product_codes = pd.DataFrame(product_grouped['Product'].unique(), index=product_grouped['Product_Code'].unique(), columns=['Code'])
dat = [dict(
    type = 'choropleth',
    locations = product_grouped['State'],
    z =  product_grouped['Product_Code'],
    locationmode = 'USA-states',
    text = product_grouped['Product'],
    colorscale = [[0.0, 'rgb(255,255,179)'], [0.25, 'rgb(141,211,199)'], [0.50, 'rgb(190,186,218)'],[0.75, 'rgb(251,128,114)'],\
            [1, 'rgb(128,177,211)']],
    colorbar = dict(
        tickmode = 'array',
        tickvals = list(product_codes.index),
        ticktext = list(product_codes.Code))
    )]

layout = dict(
    geo = dict(
        scope='usa', showlakes=True, lakecolor = 'rgb(255, 255, 255)'))

fig = dict(data=dat, layout=layout)

iplot(fig, filename='ConsumerComplaints_MostComplainedProductByState')

#!/bin/bash
import os
os.system(command)

python3.8 ./SRC/InsightCodingChallenge.py ./input/dataIn

python3.8 ./SRC/InsightCodingChallenge.py ./output/dataOut
