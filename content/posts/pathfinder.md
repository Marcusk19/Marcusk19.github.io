---
title: "PathFinder Project"
date: 2022-07-14T19:35:24-04:00
draft: true
---

For my senior design class we had to come up with a product and prototype it over the course of two semesters. My team's idea was a heads up display (HUD) that could project directions and vehicle metrics onto a driver's windshield. During the protoype phase, me and another group mate were tasked with working on the code for our product (called the PathFinder). This article goes over the software portion of our project and more in depth regarding my work.



## Hardware

We chose to use the Raspberry Pi as the board for our project. We knew that it would be easier to deal with than a microcontroller since I had previous experience working with a Pi. Python was our go to language for development since it allowed us to rapidly prototype without getting stuck on low-level issues like memory management. 



In addition to the Pi, we used several add-ons to gain more functionality:

* [PiJuice hat](https://uk.pi-supply.com/products/pijuice-standard) for portable battery when on the road.

* [UCTRONICS OLED display](https://www.amazon.com/UCTRONICS-SSD1306-Self-Luminous-Display-Raspberry/dp/B072Q2X2LL/ref=sr_1_6?crid=VZ2U5OBHC5IC&keywords=oled+display&qid=1657842254&sprefix=oled+display%2Caps%2C79&sr=8-6) for showing directions and data.

* [GPS module](https://www.amazon.com/Microcontroller-Compatible-Sensitivity-Navigation-Positioning/dp/B07P8YMVNT/ref=sr_1_2?crid=3F2AXAO2GOLR1&keywords=gps+pi+module&qid=1657842307&sprefix=gps+pi+module%2Caps%2C91&sr=8-2) for keeping track of our current position.

* [Bluetooth OBD2 scanner](https://www.amazon.com/Veepeak-Bluetooth-Diagnostic-Supports-DashCommand/dp/B011NSX27A/ref=sr_1_6?crid=8OXL7062KUQ3&keywords=obd2+scanner+bluetooth&qid=1657842344&sprefix=obd2+scanner+bluetooth%2Caps%2C95&sr=8-6) for obtaining vehicle metrics.



## Directions

The first part I tackled was obtaining directions. From my [Place Picker](https://marcuskok.com/posts/placepicker/) project I knew that Google had an API to obtain directions from point A to point B. I decided to try this approach because it was the first thing that came to my mind. Gathering the directions was straightforward. I just had to format a HTTP request to their API and parse through the returned JSON to get a list of instructions. I created a `directions.py` module and wrote a method to do the following:

```python
 def getDirections(self):
        """ Makes an HTTP request to the Google API endpoint and parses through
        the JSON response to obtain a list of instructions.
        Returns:
            string: Text output of json response.
        """
        url = 'https://maps.googleapis.com/maps/api/directions/json' # endpoint for Google API
        # define parameters for http request
        params = dict(
            origin = self.user_origin,
            destination = self.user_destination,
            key = api_key
        )

        resp = requests.get(url = url, params = params) # response from Google is loaded into resp
        directions = json.loads(resp.text) # using a json library to format Google's respone for parsing
        # print(directions)
        routes = directions['routes']

        # Parse json data looking for instructions and maneuvers
        for route in routes:
            legs = route['legs']
            for leg in legs:
                steps = leg['steps']
                for step in steps:
                    # instructions from json are in html format
                    # parsing using BeautifulSoup is required for readable text
                    ins_html = BeautifulSoup(step['html_instructions'], 'html.parser') 
                    instruction = ins_html.get_text()
                    coordinate = step['start_location']
                    self.coordinate_queue.append(coordinate)
                    self.instruction_queue.append(instruction)

        
        return directions
```

Note that instead of hard coding an `origin` and `destination` I instead passed them in as parameters. This was because my next step was to make sure that end users had an easy way to provide their two locations to our device...



## MQTT Protocol

When pitching our idea earlier, one of the selling points for the product was that it would not require the user to download an additional app to their phone to pair with the device. Looking back, that promise made it harder for us to think of a way we could incorporate a user interface into our device. Eventually, my research led me to MQTT as the solution. I chose it for several reasons:

1. MQTT has a reputation for being lightweight and reliable, two important attributes when considering the product may be used in areas with low network coverage.

2. The scalability of the protocol allowed us to claim that we could easily increase production if we every hypothetically "went to market".

3. I could create a webpage serving as a MQTT publisher that people could go to and send their locations to the PathFinder (technically not an app).



After learning about MQTT, setting up a couple things, and doing a lot of testing, I came up with a simple Flask application that would publish data to an MQTT broker. 

```python
# main.py
load_dotenv()
MQTT_SERVER = os.getenv("MQTT_SERVER") # ip address of linode instance
print("Server: " + MQTT_SERVER)

app = Flask(__name__)

@app.route('/', methods = ["POST", "GET"])
@mobile_template('{mobile/}index.html')
def index(template):
    if request.method == "POST":
        destination = request.form.get("dest")
        source = request.form.get("source")
        publish.single("destination", destination, hostname=MQTT_SERVER)
        publish.single("source", source, hostname=MQTT_SERVER)
        print("Published message to " + MQTT_SERVER)    
    return render_template(template)

if __name__ == "__main__":
    from waitress import serve
    serve(app, host="0.0.0.0", port="80")
```

```html
<!-- index.html -->
<!DOCTYPE html>
<head>
    <h1 id="title">PathFinder</h1>
    <link rel="stylesheet" type="text/css" href="{{ url_for('static', filename= 'css/stylesheet.css') }}" />
</head>


<body>
    <div>
        <form method="POST" id="input">
            <input type="text" placeholder="Enter Current Location" name="source" class="feedback-input" />
            <input type="text" placeholder="Enter Destination" name="dest" class="feedback-input" />
            <input type="submit" placeholder="Submit" />
        </form>
    </div>
</body>
```



I dockerized this app and put it on a public cloud instance (Linode) alongside a MQTT broker container (Eclipse Mosquitto). After that I had to create a `HUD.py` module, and write some methods to subscribe to the broker. Below is a snippet from that file as an example:

```python
def on_source(self, client, userdata, msg):
        """ Defines what should happen upon receiving a message published
        on the 'source' channel.
        Formats message payload and saves to local variable inputA.
        Args:
            client (client): MQTT client class instance.
            userdata (userdata): A user provided object passed to the on_message callback.
            msg (msg): Message class from the broker.
        """
        self.inputA = str(msg.payload)
        self.inputA = self.inputA[2 : len(self.inputA)-1] # get rid of mqtt formatting
        print("inputA = " + self.inputA)

    def on_dest(self, client, userdata, msg):
        """ Defines what should happen upon receiving a message published
        on the 'destination' channel.
        Formats message payload anad saves to local variable inputB.
        Args:
            client (client): MQTT client class instance.
            userdata (userdata): A user provided object passed to the on_message callback when a message is received.
            msg (msg): Message class from the broker.
        """
        self.inputB = str(msg.payload)
        self.inputB = self.inputB[2 : len(self.inputB)-1] # get rid of mqtt formatting
        print("inputB = " + self.inputB)
```

I created a diagram to better convey what is going on for our documentation. The user goes to our webpage on their device, fills out the fields, and the PathFinder can obtain the information from the broker.

![](https://github.com/Marcusk19/PathFinder/raw/main/readme_images/mqtt.jpg)

Here's what our page looked like. Since the class ended it's no longer up considering I didn't want to keep paying to host it in the cloud.

![](https://github.com/Marcusk19/PathFinder/raw/main/readme_images/website.png)

## OBD

OBD functionality was mostly handled by my partner. I won't go into a lot of detail here, but I can say it did take some fiddling with the code to even get a connection to our bluetooth reader. 
