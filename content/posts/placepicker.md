---
title: "Place Picker"
date: 2021-12-26T08:47:11+01:00
draft: false
---
Something I always thought was cool about programming was the high-level functionality you could build into programs using other developers’ APIs. Coming from an ECE background however, I didn’t have much chance to mess around with that kind of stuff. This project was a way of getting my feet wet in working in API calls with my programs and starting to work in Python – which ended up to be much more fun than I thought.

## The Logic

![](https://marcuskok.tech/wp-content/uploads/2022/01/My-First-Board-1024x372.jpg)

_Simple diagram of how an API works, basically I can make a request over the internet to a web service and they can provide me with a response – this way Google can take care of all the hard work for me_

As always when I start a new project, I go to Google to help me get started. Much of the credit for this project goes to Regita H. Zakia, the writer of this article [here](https://towardsdatascience.com/foods-around-me-google-maps-data-scraping-with-python-google-colab-588986c63db3) that I found. Reading through it gave me a better understanding of how I could get the information I wanted from Google’s API – and specifically how I could parse through the JSON response to get a name, address, and rating of restaurants nearby.

Where my project differed from the one in the article above was that I wanted a simple program that could automatically gather places around me into a list without any prompting from the user and then randomly display one. In other words if I had trouble deciding on a place to eat at I could pull out my app and have somewhere suggest a location for me.

To solve the problem of automatically grabbing locations for me I decided to use IpInfo’s API to get a location for me so I could then pass that location to Google’s API. More information on IpInfo can be found [here](https://ipinfo.io/). In order to simplify things (and force myself to work across multiple files) I broke up the logic into two .py files – ipHandler.py and placePicker.py.

The ipHandler.py file had only one job, and that was to return a city and location from the current IP address.

Documentation from IpInfo was really straightforward and I was able to accomplish this with only five lines (with a commented sixth line for my own debugging purposes)

```
access_token = picker.config.access_token
handler = ipinfo.getHandler(access_token)
ip_address = get('https://api.ipify.org').content.decode('utf8')
# print('ip: {0}'.format(ip_address)) // Printing out the ip to make sure I'm doing this right
details = handler.getDetails(ip_address)
```

Then all I had to do was build two simple functions to return the location from the details object:

```
def get_location():
    return details.loc

def get_city():
    return details.city
```

Moving onto placePicker.py, this is where I pass in a city for my request to the Google Places API from ipHandler.py by calling the get\_city() function I just wrote. Then it’s a simple matter of parsing the JSON response and putting all the restaurants into an array. After that I choose a place using a randomly generated index of the array and print it to the console. I only care about the name, address, and location, so those are the values I print from the places object obtained from the JSON. Information like latitude and longitude can stay hidden from the end user.

```
def pickPlace():
    # declare an array of places to choose from
    all_places = []
    keywords = ['restaurant']
    api_key = config.api_key
    radius = '1000'
    coordinates = ipHandler.get_location() # obtain coords from ip addr
    city = ipHandler.get_city()
    print('Searching {0} within radius {1}...'.format(city, radius))
    # print(coordinates)

    for keyword in keywords:
        url = 'https://maps.googleapis.com/maps/api/place/nearbysearch/json?location='+coordinates+'&radius='+str(radius)+'&keyword='+str(keyword)+'&key='+str(api_key)
        while(True):
        # print(url)
            respon = requests.get(url)
            jj = json.loads(respon.text)
            results = jj['results']
            for result in results:
                name = result['name']
                place_id = result['place_id']
                lat = result['geometry']['location']['lat']
                lng = result['geometry']['location']['lng']
                rating = result['rating']
                types = result['types']
                vicinity = result['vicinity']
                place = [name, place_id, lat, lng, rating, types, vicinity]
                all_places.append(place)

            if 'next_page_token' not in jj:
                break
            else:
                next_page_token = jj['next_page_token']
                url = 'https://maps.googleapis.com/maps/api/place/nearbysearch/json?key='+str(api_key)+'&pagetoken='+str(next_page_token)
                continue
        labels = ['Place Name', 'Place ID', 'Latitude', 'Longitude', 'Rating', 'Types', 'Vicinity']
            #    [    0            1            2           3           4          5         6    ]
        
        # code that exports data to csv file if needed
        # export_dataframe_1_medium = pd.DataFrame.from_records(all_places, columns=labels)
        # export_dataframe_1_medium.to_csv('export_dataframe_1_medium.csv')
        
    x = 0
    y = len(all_places) - 1
    random_index = random.randint(x,y)

    name = all_places[random_index][0]
    rating = all_places[random_index][4]
    address = all_places[random_index][6]
    time.sleep(3)
    # print('I suggest "{0}" with a rating of {1} at "{2}"'.format(name, rating, address))
    return "I suggest " + str(name) + " with a rating of " + str(rating) + " at " + str(address)
```

## The UI

With most of the background work done, I then started work on building a UI for my application. At the time I was thinking it would be neat to put this on an iPhone which led me to BeeWare, a set of tools to write cross platform GUI applications that you can find [here](https://beeware.org/ "https://docs.beeware.org/en/latest/index.html#").

Now to be honest, building user interfaces isn’t really my strong suit so I started out easy with the goal of building a simple window with a button that the user can press to suggest a place. BeeWare has some great documentation and I made use of their [getting started guide](https://docs.beeware.org/en/latest/index.html#). I was able to get a very rudimentary UI going with quite literally only a button to push that calls the pickPlace() method from placePicker.py.

```
class Picker(toga.App):

    def startup(self):
        """
        Construct and show the Toga application.

        Usually, you would add your application to a main content box.
        We then create a main window (with a name matching the app), and
        show the main window.
        """
        main_box = toga.Box()
        button = toga.Button('Choose place', on_press=self.button_handler)
        button.style.padding = 50
        button.style.flex = 1
        main_box.add(button)
        self.main_window = toga.MainWindow(title=self.formal_name)
        self.main_window.content = main_box
        self.main_window.show()

    def button_handler(self, widget):
        # placePicker.hello()
        self.main_window.info_dialog(
            placePicker.pickPlace(),
            'Thank you!'
        )
```

![](https://marcuskok.tech/wp-content/uploads/2022/01/Screen-Shot-2022-01-21-at-10.05.02-PM.png)

_My very simple interface_

![](https://marcuskok.tech/wp-content/uploads/2022/01/Screen-Shot-2022-01-21-at-10.06.18-PM.png)

_Pop up works as intended!_

For now this is where my development of the project ends. I ran into some issues when trying to package the app for distribution and just couldn’t get around it. I’d definitely like to return to it and polish up this project in the future, but more immediate deadlines are looming on the horizon. This was a fun little project for me to work on during an afternoon and was a great learning experience. If you would like to check out the source code for this project, it’s on [GitHub](https://github.com/Marcusk19/Place-Picker).
