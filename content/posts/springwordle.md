---
title: "Spring Boot Wordle"
date: 2022-03-10T21:57:09+00:00
draft: false
---
I’ve wanted to build a web-app using Java and the recently popular game “Wordle” looked easy to replicate in my own way. The original Wordle game itself was only a Javascript file for most of the logic. However, I took this as a chance to learn some basics of Spring Boot and build a proper web-application based on the game.

![](https://marcuskok.tech/wp-content/uploads/2022/02/Screen-Shot-2022-02-27-at-4.14.16-PM.png)

File structure of my project

##### The Wordle Class

Getting started was simple enough thanks to the [Spring Initializr](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-spring-initializr "Spring Initializr") extension installed on my VScode. Spring also provides [their own initializer](https://start.spring.io/ "their own initializer") as an alternative. I chose Gradle over Maven mainly because I had used Maven previously from messing around with Spring and wanted to try Gradle out for a change. I’m still not clear on the pros and cons for one or the other so I’ll probably come back and edit this more when I have a better understanding.

The first step to working out how I was going to implement this project was to think about how the logic behind the game would work. I decided a good approach was to create a Wordle class as so:

```java
public class Wordle {

    List <String> words = new ArrayList<String>(); // a list of words to pick from randomly
    private String solution = "marky"; // variable to hold the solution
    private int numGuesses = 0; // counter for the number of guesses the player has
    private boolean correctGuess = false; // flag for when player makes the correct guess

}
```

There are two different ways to fill up the words list that I implemented. Initially I had a simple .txt file with all the five letter words – totaling roughly 1,000 lines. It was fine to read from that file when I was testing my code locally. However, when I tried packaging it up using Gradle I ran into issues because the packaged code couldn’t find my .txt file.

As a workaround I decided to pull directly from where I got the list of words from: GitHub. I found some example code of how to connect and read raw data from GitHub which seemed to do the trick.

```java
try { 
            url = new URL("https://raw.githubusercontent.com/charlesreid1/five-letter-words/master/sgb-words.txt");
            URLConnection uc = url.openConnection();
            uc.setRequestProperty("X-Requested-With", "Curl");
            String userpass = username + ":" + password;
            String basicAuth = "Basic" + new String(Base64.getEncoder().encode(userpass.getBytes()));
            uc.setRequestProperty("Authorization", basicAuth);

            BufferedReader reader = new BufferedReader(new InputStreamReader(uc.getInputStream()));
            String line = null;
            while ((line = reader.readLine()) != null) 
                words.add(line);
            
            reader.close();
        } catch(Exception e) {
            e.printStackTrace();
        }
```

Then I had an idea to set up a MySQL server in the cloud and store my data there. It derailed me for a bit while I tried to figure out how to get stuff to work, but I ended up learning some useful stuff for future projects. Instead of getting data from GitHub this code reaches out to a database I already pre-populated with all the words. To make sure I wasn’t leaking any sensitive credentials, I made use of a properties file to define my JDBC url along with username and password.

```java
        try {
            Properties prop = new Properties();
            InputStream input = getClass().getClassLoader().getResourceAsStream("config.properties");

            if(input != null) {
                prop.load(input);
            } else {
                System.out.println("Properties file not found");
            }

            url = prop.getProperty("url");
            username = prop.getProperty("username");
            password = prop.getProperty("password");
        } catch(IOException ioe) {
            ioe.printStackTrace();
        }


        
        // connection to mysql database
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
            Connection con = DriverManager.getConnection(url, username, password);
            Statement stmt = con.createStatement();
            ResultSet rs = stmt.executeQuery("select * from five_letters");
            // add all results from result set to words array
            while(rs.next()) words.add(rs.getString(1)); 
            con.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
```

Moving forward I went with the GitHub approach; simply because it ended up being easier in the long run and I didn’t want to spend money on a dedicated node in the cloud for a MySQL database.

Now it was time to define the methods for the game. To start with here’s the implementation for choosing a solution from the list of words at random. Basically it uses a random int as the index from the list to choose from.

```java
public String chooseSolution() {
        // select a random word from words to use as the solution
        if(words.size() <= 0)  return "No list"; // don't do anything if we can't fetch from the db
        Random rnd = new Random();
        int index = rnd.nextInt(words.size());
        this.solution = words.get(index);
        numGuesses = 0;
        return "Solution chosen";
    }
```

The biggest part is validating a user provided guess against the solution. Several considerations have to be made. For instance, whether the user entered a valid word or not. Additionally, once a letter has been determined to be in the correct spot, it should no longer be considered for checking or it could lead to false positives if the guess contains multiple of that letter. We also keep track of the number of guesses the user has entered along with whether the correct guess has been flagged or not. For each letter in the guess there are three possibilities:

-   “correct” – letter is in the correct space.
-   “present” – letter is **not** in the correct space but is present somewhere else in the solution.
-   “absent” – letter is not present anywhere in the solution.
    
    The function returns an array of strings. Each one represents the state of its respective letter in the guess.  
    For example, a guess of “plane” for the solution “crane” would return \[“absent”, “absent”, “correct”, “correct”, “correct”\]
    

```java
public String[] validateGuess(String guess) {
        guess = guess.toLowerCase();
        numGuesses++;
        String validation[] = new String[solution.length()];
        String tempSol = solution;
        int correct = 0;

        // check for valid input
        if(!(words.contains(guess)) || guess.length() < solution.length()) {
            validation[0] = "invalid";
            if(numGuesses > 0) numGuesses--;
            return validation;
        }

        // check for correctness
        for(int i = 0; i < guess.length(); i++) {
            char letter = guess.charAt(i);
            if(letter == tempSol.charAt(i)) {
                // add correct
                correct += 1;
                validation[i] = "correct";
                // remove correct char from tempSol so is not doubly checked
                tempSol = tempSol.substring(0, i) + "-" + tempSol.substring(i+1, tempSol.length());

            }
            else if(tempSol.contains(Character.toString(letter))) {
                // add present 
                validation[i] = "present";
            } else {
                // add absent
                validation[i] = "absent";
            }
        }
        if(correct == solution.length()) {
            correctGuess = true;
        }
        return validation;    
    }
```

After the validation method all that remains to do is build some helper functions to access important instance variables of the class.

```java
    public List<String> getWords() {
        // handler function to return words array
        return words;
    }

    public String getSolution(){
        // handler funciton to return solution
        return this.solution.toUpperCase();
    }

    public String checkGameOver() {
        if(correctGuess || numGuesses >= 6) return "true";
        else return "false";
    }
```

##### Controllers and Applications

This may not be the most optimized way to do things, but I decided to build two Spring Boot controllers. One that handles API calls for the Java backend and another to serve the index.html file for the frontend.

Additionally, I added some endpoints to check the current solution and status of the game that were helpful while debugging.

```java
@RestController
@RequestMapping(value = "/api")
public class WordleController {
    Wordle wordle = new Wordle();

    @GetMapping("/solution")
    public String solution() {
        return wordle.getSolution();
    }

    @GetMapping("/validate")
    public String[] validate(@RequestParam(value="guess", defaultValue="     ")String guess) {
        return wordle.validateGuess(guess);
    }

    @GetMapping("/check_game")
    public String checkGame() {
        return wordle.checkGameOver();
    }

    @GetMapping("/choose_solution")
    public String choose_solution() {
        return wordle.chooseSolution();
    }
}
```

```java
@Controller
public class SpringWordleApplicationController {

    @RequestMapping("/")
    public String index() {
        return "index";
    }

}
```

Nothing too special in the application file, just a main function to start it.

```java
@SpringBootApplication
@EnableAutoConfiguration
@ComponentScan
public class SpringWordleApplication {

public static void main(String[] args) {
SpringApplication.run(SpringWordleApplication.class, args);
}

}@SpringBootApplication
@EnableAutoConfiguration
@ComponentScan
public class SpringWordleApplication {

public static void main(String[] args) {
SpringApplication.run(SpringWordleApplication.class, args);
}

}
```

##### The Frontend

From looking at other approaches to coding Wordle it seemed like a lot of people chose to automate the process of building the grid with Javascript.

```js
// create the board
    for(let r = 0; r < height; r++) {
        for(let c = 0; c < width; c++) {
            let tile = document.createElement("span");
            tile.id = r.toString() + "-" + c.toString();
            tile.classList.add("box")
            tile.innerText = "";

            document.getElementById("board").appendChild(tile);
        }
    }
```

Then event listeners are added for user entered key-presses. It’s important to add a listener event for the backspace key so players can edit their guesses.

```js
document.addEventListener("keyup", (e) => {
        if(gameOver) return;
        // alert(e.code)
        if("KeyA" <= e.code  && e.code <= "KeyZ"){
            if(col < width){
                let currTile = document.getElementById(row.toString() + '-' + col.toString());
                if(currTile.innerText == "") {
                    currTile.innerText = e.code[3];
                    col += 1;
                }
            }
        }
        else if(e.code == "Backspace") {
            if(0 < col && col <= width) {
                col -= 1;
                let currTile = document.getElementById(row.toString() + '-' + col.toString());
                currTile.innerText = "";
            }
        }
        else if(e.code == "Enter") {
            check();
            if(row < height) row += 1;
            col = 0;
        }

    })
```

Every time the user presses the Enter key the check() function is called. The logic behind this function turned out to be more heavy than I thought it would, but it does the job. A few things are done within the check() function. First, it reaches out to the backend and checks the status of the game. It then reaches out again and validates the user’s guess, then updates the colors of the each box according to the validation array returned. Lastly, it updates the current row the user is on so they can make their next guess.

```java
function check(){
    // check if game is over
    fetch(apiUrl + '/check_game')
    .then(response => {
        return response.text();
    })
    .then((data) => {
        let endGame = data;
        console.log(endGame);
        
        if(endGame == "true") {
            // only get solution if the game is over
            fetch(apiUrl + '/solution')
            .then(response => {
                return response.text();
            })
            .then((text) => {
                let solution = text;
                document.getElementById("answer").innerText = solution;
            })

            gameOver = true;
            return;
        }
    })

    var guess = ""
    for(let c = 0; c < width; c++) {
        let currTile = document.getElementById(row.toString() + '-' + c.toString());
        let letter = currTile.innerText;
        guess = guess + letter;
    }
    console.log(guess);

    fetch(apiUrl + '/validate?guess=' + guess)
    .then(response => {
        return response.json();
    })
    .then((data) => {
        let validate_arr = data;

        for(let i = 0; i < width; i++) {
            console.log(validate_arr[i]);
            let currTile = document.getElementById((row-1).toString() + '-' + i.toString());
            console.log(row.toString() + '-' + i.toString());
            if(validate_arr[0] == "invalid") {
                currTile.innerText = "";
            } else {
                currTile.classList.add(validate_arr[i]);
            }
        }
        if(validate_arr[0] == "invalid") row -= 1;
    })
}
```

The index.html file is pretty simple compared to the Javascript. It defines the header for the UI along with a div to contain the game board.

```html
<!DOCTYPE html>
<html>
    <head>
        <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm" crossorigin="anonymous">
        <link href="css/index.css" rel="stylesheet" />
    </head>
    <body>
        <h1 id="title">Poor Man's Wordle</h1>
        <br>
        <div id="board">
            <script src="js/wordle.js"></script>
        </div>
        <br>
        <h1 id="answer"></h1>
        
    </body>
</html>
```

All that’s left after putting everything together is to make the executable with `gradle clean build`  
and then run it with `java -jar build/libs/spring_wordle-1.1.jar`

![](https://marcuskok.tech/wp-content/uploads/2022/02/Screen-Shot-2022-02-27-at-3.59.26-PM.png)

The final result

##### Conclusion

I learned several things through doing this project. Firstly, when I started making it I got caught up with putting a lot of the logic on the frontend Javascript file. When I brought this up with my friends who had more knowledge about this sort of stuff, they suggested that I instead try building a REST API and have the frontend make calls to it. This led me to the approach shown here. I also learned that each call should be independent of each other. In other words, there shouldn’t be a required order that these calls have to happen in – which led me to rethink how I wanted the Wordle.java logic to behave. Beyond just learning about building an API, I also discovered how to programmatically make HTTP requests in both Java and Javascript. The end result may be far from perfect as the official Wordle (As of writing this I still need to add a keyboard to let players keep track of the letters they use). However, this was a fun project to work on between doing the daily wordles.

If you would like to check out the full code for the project, it’s all on my GitHub [here](https://github.com/Marcusk19/Spring_Wordle "here").
