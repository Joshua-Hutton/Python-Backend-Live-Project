# Python-Backend-Live-Project

Python Backend 
For this sprint I work on a python Django project. This project Name Travalscrap was a website for planning a trip. In it we provided services one might need for planning a trip. I came in to this in the middle of the project after working with C# for about a month a half and had to re familiarize myself with python.  

The first user story I worked on was a currency conversion app. The app already exist within the program. The issue was the API the app used to convert the currency. The whole thing was based on euros and the user wanted strait conversion from one currency to another. In the description of the story a new API was already decided on and all I had to do was rework it to work with this new API. 

This is the code for the new app view I created. 
```python
import json
import urllib.request as urllib

from django.shortcuts import render
from .forms import CurrencyForm


# Create your views here.

#The API: https://www.currencyconverterapi.com/docs
def currency(request):
  
   apiKey = 'd574e8c933cc8c93eda7'

   if request.method == "POST":
       form = CurrencyForm(request.POST)
       if form.is_valid():

           target = form.cleaned_data['target_currency']
           base = form.cleaned_data['base_currency']
           imput = form.cleaned_data['money']
           query = base + "_" + target
           url = 'https://free.currconv.com/api/v7/convert?q=' + query + '&compact=ultra&apiKey=' + apiKey
          
        
           j = json.loads(urllib.urlopen(url).read())
          
           amount = '{0:.2f}'.format(imput * j[query])
           original = '{0:.2f}'.format(imput)
          
           conversion = {
               'original' : original,
               'amount' : amount,
               'currency' : target,
               'base' : base,
           }
          
           context = {'conversion' : conversion}
           return render(request, 'currency/currency_conversion.html', context)
   else:
       form = CurrencyForm(request.POST)
       return render(request, 'currency/currency.html', {'form' : form})
```

This is the original code.
```python
import json
import urllib

from django.shortcuts import render
from .forms import CurrencyForm


# Create your views here.

# fixer.io API key : 2279dc03455788be16c0da66470aea36 limit of 1000 uses/month and base currency of EUR
def currency(request):
   url = 'http://data.fixer.io/api/latest?access_key=2279dc03455788be16c0da66470aea36'

   if request.method == "POST":
       form = CurrencyForm(request.POST)
       if form.is_valid():
           j = json.loads(urllib.request.urlopen(url).read())
           target = form.cleaned_data['target_currency']
           base = form.cleaned_data['base_currency']
           dollars = form.cleaned_data['money'] # Dollars, our base currency
           euros = dollars / j["rates"][base] # Convert to Euros, the API's base currency
           c = j["rates"][target]
           original = '{:,.2f}'.format(dollars)
           amount = '{:,.2f}'.format(round(euros * c, 2)) #Format to display money
           conversion = {
               'original' : original,
               'amount' : amount,
               'currency' : target,
               'base' : base,
               'rates' : j["rates"]
           }
          
           context = {'conversion' : conversion}
           return render(request, 'currency/currency_conversion.html', context)
   else:
       form = CurrencyForm(request.POST)
       return render(request, 'currency/currency.html', {'form' : form})
```
This was the bulk of the work and any other changes to other files in the program were minor. 

The next story I took was to create a new app in the program. This app would take in either the name of a city that the user wanted to find an airport in or the IATA 3 character code of an airport and return the city it could be found in or a list of airports near that city depending on the input. . this project took a large part of my two weeks due to having issues with API I selected. I stuck with this API becuase I not only took in the input i needed it to but also was being used elswar in the project to gather hotel information. After figuring out some mistakes I made and the API’s  servers coming back online I was able to figure out the project. 

Here is the code for that project. 

The form. 
```python
from django import forms
forms.py
class AirportForm(forms.Form):
  search = forms.CharField(required=False)
  response = forms.CharField(required=False)
  airport = forms.CharField(required=False)
```
The first templete. 

airport.html
```python
{% extends "./AirportApp_base.html" %}
{% block content %}
{% include "nav.html" %}
<h1 class='text-center pt-5'>Airport finder</h1>
<div class="airportForm">
<form method="GET">
{% csrf_token %}
<p>Looking for an airport?</p>
<p>Please Enter either the 3 character airport code or city name you traveling to.</p>
<!--Search works on city name keyword or on IATA code. If IATA is entered you will receive information on the corisponding airport.
If a city name is entered it take the input and return where it finds that word. Meaning if you enter "Jacksonville" you will get airport infomation for both
Jacksonville, Florida as well as Jacksonville, North Carolina. If you enter "Detriot" you will get records for both Detriot, Michagan and
Detriot Lake, Minisoda.-->
{{ form.search }}
<br>
<input type="submit" value="Submit" style="margin: 5px;">
</form>
</div>
{% endblock content %}
```
The template for the API response. 
airport_results.html
```python
{% extends "./AirportApp_base.html" %}
{% block content %}
{% include "nav.html" %}
<h1 class='text-center pt-5'>Airport Finder</h1>
<br />
<div class="airportResults" >
<h3><a href="/AirportApp/">←&nbsp;Back</a></h3>
<p>Your Search: {{ results.search }}</p> <!--displays the search input from user-->
{% for result in results.response %}<!--Loops through list from json-->
{% if result.subType == 'AIRPORT' %}<!--List contains info on cities that is filtered out by if statment-->
<hr>
<p>IATA: {{ result.iataCode }}</p>
<p>Name: {{ result.name }}</p>
<p>City: {{ result.address.cityName }}</p>
<p>State: {{ result.address.stateCode }}</p>
<p>Country: {{ result.address.countryCode }}</p>
{% endif %}
{% endfor %}
</div>
{% endblock content %}
```
The view.

Views.py
```python
from django.shortcuts import render
from .forms import AirportForm
from amadeus import Client, Response
from amadeus import Location
# Create your views here.
# https://github.com/amadeus4dev/amadeus-python <--- Info for the amadeus SKD
# The Api returns a json based on the search input. If the search input is an IATA code it will return a single listing for
# that airport. If search is a city name it will return the first 10 results that match the search word. The records in the
# Api are sorted based on a consumer feedback rating so they will not be an alphabetic order. I do not know where the ratings
# came from. More information on Amadeus --> https://developers.amadeus.com/self-service/category/air/api-doc/airport-and-city-search
def airport(request):
  if request.method == "GET":
    form = AirportForm(request.GET)
    if form.is_valid():
    
      search = form.cleaned_data['search']
      response = form.cleaned_data['response']
      
      amadeus = Client( # Athentication for the Amadeus API
        client_id = "oXoHcPGNhQAKNcvvhFIkB9kFudwrBYTy",
        client_secret = "zs3PhgM4HNpZbCm4"
      )
      
      if form.data.get('search'): #Checks to see if there is input in the form
        response = amadeus.reference_data.locations.get( #retrieves the response from the api as a json
          keyword=search,
          subType=Location.ANY
        )
      response = response.data #Extracts list of locations from the json to be sent to airport_results
      
      results = {
        'response' : response,
        'search' : search,
      }
      context = {'results' : results}
      return render(request, 'airport/airport_results.html', context)
    else:
      form = AirportForm(request.POST)
      return render(request, 'airport/airport.html', {'form' : form})
```
The segment from the site css file for the templates.
This was styled after the rest of the site.
```css
/*****************************************/
/* CSS PAGE STYLES FOR AirportApp */
/*****************************************/
/* Note: Please use a unique naming convention when targetting element for each and every APP.
This is to eliminate styles stepping over one another.
Good example: check Travel Advisory! */
.airportForm{
text-align: center;
width: 40%;
margin: 0 auto;
padding: 10px;
background-color: lightgray;
border: 6px outset #9a9;
box-shadow: 5px 5px 15px rgba(0,0,0,0.4);
-webkit-box-shadow: 5px 5px 15px rgba(0,0,0,0.4);
-moz-box-shadow: 5px 5px 15px rgba(0,0,0,0.4);
}
.airportResults{
text-align: center;
width: 40%;
margin: 0 auto;
padding: 10px;
background-color: lightgray;
border: 6px outset #9a9;
box-shadow: 5px 5px 15px rgba(0,0,0,0.4);
-webkit-box-shadow: 5px 5px 15px rgba(0,0,0,0.4);
-moz-box-shadow: 5px 5px 15px rgba(0,0,0,0.4);
}
```
