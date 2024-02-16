
# Ice Hockey Fantasy App | Backend Integration

## Project Summary:

The developer was tasked with integrating an agile [Airtable](https://airtable.com/) backend with a partially completed [Flutter](https://flutter.dev/) front-end app for a non-profit Fantasy Ice Hockey League in England.  

This project involved approaches related to data management and user authentication. The client sought a cost-effective solution considering an initial limited audience size and the developer crafted his approach accordingly.

## Project Approach:

Data modeling was executed using a model/service implementation to maintain a structured [MVVM](https://medium.com/@onurcem.isik/introduction-to-mvvm-architecture-5c5558c3679) architecture. Dynamic API service classes were created to handle interacting with the API to enforce a separation of concerns.


### Use of Generics for Service APIs

Generics were used within the *Top Scorers* models to create a singular service for various Airtable [JSON](https://www.w3schools.com/whatis/whatis_json.asp) models. 

A simple base abstract class called `TopScorerModel` was crafted [Dart generics](https://dart.dev/language/generics), while recursive functions enabled the consolidation of paginated table data into a unified list. 

For example, the `TopOverallScorer` model implemented the `TopScorerModel` as exemplified below:
~~~dart
class TopOverallScorer implements TopScorerModel {

    String?  name;
    String?  positions;
    String?  team;
    String?  nationality;
    int?  goals;
    //...
  
    TopOverallScorer(
    {     this.name,
        this.positions,
        this.team,
        this.nationality,
        this.goals, //...
    });

    TopOverallScorer.fromJson(Map<String, dynamic> json) {
        name = json['Name'];
        positions = json['Positions'];
        team = json['Team'];
        nationality = json['Nationality'];
        goals = json['Goals'];
        // ...
    }

    @override
    Map<String, dynamic> toJson() {
        final  Map<String, dynamic> data  =  <String, dynamic>{};
        data['Name'] = name;
        data['Positions'] = positions;
        data['Team'] = team;
        data['Nationality'] = nationality;
        data['Goals'] = goals;
        // ...
        return  data;
    }

    // public access methods
    String?  getAbbrevTeamName() {
        if (team == null) return  null;
        List<String> splitName = team!.split(' ');
        if (splitName.length >= 2) {
            return splitName[1];
        } else {
            return team;
        }
    }

    // further helper functions...
}
~~~
  
This service was consumed through an [asynchronous](https://dart.dev/codelabs/async-await) service class that used Dart generics to specify which model should be downloaded through the following syntax `TopStandingsServiceAPI.getModel<TopOverallScorer>()` (ex. for the `TopOverallScorer` model).  

This service also implemented pagination, environment variables, and recursive functions to successfully build a full list by model.

~~~dart
class TopStandingsServiceAPI {
  static Uri buildUri<T extends TopScorerModel>([String? pageOffset]) {
  
    //Load .env variables through dotenv package
    var baseUrl = dotenv.env['baseUrl'];
    // ...

    //Encoded string to sort by ID column in ascending order
    const sortByID = "sort%5B0%5D%5Bfield%5D=ID&sort%5B0%5D%5Bdirection%5D=asc";

    //Add the page offset if applicable
    var offsetField = "";
    if (pageOffset != null) {
      offsetField = "&offset=$pageOffset";
    }
    
    //Build the URI based on the model type passed
    switch (T) {
      case TopOverallScorer:
        return Uri.parse('$baseUrl/$baseId/$topScorersOverallId?$sortByID$offsetField');
      case TopScorerByTeamPerWeek:
        return Uri.parse('$baseUrl/$baseId/$topScorersByTeamPerWeekId?$sortByID$offsetField');
      default:
        throw Exception(
            "top_standings_service_api -> buildUri: Invalid Model Type passed");
    }
  }

  //Get the standings of the player from the Airtable API
  static Future<List<T>?> getModel<T extends TopScorerModel>() async {
    List<T>? airtableTable;

    // Load .env variable from dotenv package
    var apiKey = dotenv.env['readAPIKey'];

    dynamic getPaginatedData([String? offset]) async {
      //Build the URI based on the model type passed
      var uri = buildUri<T>(offset);
      var headers = {
        'Authorization': 'Bearer $apiKey',
        'Accept': 'Application/json',
      };

      //Await the JSON response
      final response = await http.get(uri, headers: headers);

      if (response.statusCode != 200) {
        Exception(response.body);
      } //Unsuccessful return

      //Decode the JSON
      var json = jsonDecode(response.body);

      //Add list of records
      if (json['records'] != null) {
        airtableTable ??= []; //If array is null, assign it to be initialized
        json['records'].forEach((v) {
          if (v['fields'] != null) {
            switch (T) {
              case TopOverallScorer:
                airtableTable?.add(TopOverallScorer.fromJson(v['fields']) as T);
              case TopScorerByTeamPerWeek:
                airtableTable
                    ?.add(TopScorerByTeamPerWeek.fromJson(v['fields']) as T);
              default:
                throw Exception(
                    "top_standings_service_api -> getModel: Invalid Model Type passed");
            }
          } 
        });
      }

      if (json["offset"] != null) {
        //Recursively get the next page:
        await getPaginatedData(json["offset"]);
      }
    }

    //Call recursive function
    await getPaginatedData();

    //Return list of records
    return airtableTable;
  }

~~~

### State Memory Leak Prevention
While integrating the API into the view widgets, the vanilla setState manager of Flutter was employed. 

Because of the [generational garbage collection](https://dart.googlesource.com/sdk/+/refs/tags/2.15.0-99.0.dev/runtime/docs/gc.md) in Dart, however, the garbage collection can [unintentionally be circumnavigated](https://medium.com/flutter/flutter-dont-fear-the-garbage-collector-d69b3ff1ca30) whenever an instance to self (capture in `setState(() {})` is retained after the object is deallocated.  

Issues were occurring whenever `setState(() {}` was being called on call completions of asynchronous network requests.  Specifically, these issues were relative whenever a user had switched to a different page before the network had returned a response.  Whenever the network returned a response, `self` within the widget was assumed to be strongly referenced which caused program crashes and could cause memory leaks.

To work around this issue, the `(mounted)` condition was checked before setting state to prevent memory complications similar to strong memory cycle retainment cycles in Swift.

> Note: In the future, it would be beneficial to explore other package managers such as GetX, Bloc, or Provider to help support a true MVVC project structure and mitigate limitations of vanilla state management in Flutter

### Authentication

[Google Authentication](https://firebase.google.com/docs/auth) was implemented in order to streamline user login and forgot password logic.  By integrating Google Firebase into the project including the necessary credentials, the developer was able to quickly integrate an authentication backend including capabilities such as password recovering and logging out.
 
### User Defaults

Both Android and iOS have different mechanisms for persisting user data across app sessions.  This behavior is paramount for retaining information even when the app is closed and was preferred as a method to retain the user's information across app sessions.  

The "[shared_preferences](https://pub.dev/packages/shared_preferences)" package was integrated using [pub](https://pub.dev/) to complement an implemented asynchronous service named `UserInfoService`:

~~~
class UserInfoService {
  static const _userEmailKey = "email";
  static const _userNameKey = "name";

  static Future<void> setUserEmail(String? email) async {
    final prefs = await SharedPreferences.getInstance();
    if (email != null) {
      await prefs.setString(_userEmailKey, email);
    } else {
      await prefs.remove(_userEmailKey);
    }
  }

  static Future<String?> getUserEmail() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getString(_userEmailKey);
  }

  static Future<void> setUserName(String? name) async {
    final prefs = await SharedPreferences.getInstance();
    if (name != null) {
      await prefs.setString(_userNameKey, name);
    } else {
      await prefs.remove(_userNameKey);
    }
  }

  static Future<String?> getUserName() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getString(_userNameKey);
  }

  static Future<void> saveUserName(String email) async {
    if (email == "") {
      return;
    }

    var userName = await PlayerStandingServiceAPI.getNameFromEmail(email);
    if (userName != null) {
      UserInfoService.setUserName(userName);
    } else {
      throw AuthenticationError(AuthenticationErrorType.userNotFound);
    }
  }
}
~~~

### Environment Variables
In order to comply with concepts outlined in the [12 Factor App](https://12factor.net/) guidelines, the developer decided to use environment variables instead of directly injecting API credentials into the code.  The "[flutter_dotenv](https://pub.dev/packages/flutter_dotenv)" package was integrated through `pub` to integrate this functionality.

### Tuple Integration
There were occasional service methods, due to data structure, that needed to return pre-grouped instances of the `HockeyPlayer` model.  For example, a compound return was necessary when a list of players *and* a captain had to be returned from one method.  

To provide clarity to these return types and not simply use vague `List<String, dynamic>` clauses,  the developer integrated the "[tuple](https://pub.dev/packages/tuple)" package to use tuples to structure compound return types from methods.

### Display Updates

While integrating the backend, there were necessary design updates that had to be made.  Additionally, the developer was tasked with integrating some additional views.  A few of the visual updates are:

#### Skeletonizer
*Using the "[skeletonizer](https://pub.dev/packages/skeletonizer)" `pub` package to provide crisp loading indicators while data was loading*

<img src="https://joshzbeck.com/wp-content/uploads/2024/02/Standings-Loading.png" alt="Trade Screen (Loading)" width="150"/> 

> "Standings" view (loading with skeleton)

<img src="https://joshzbeck.com/wp-content/uploads/2024/02/Standings.png" alt="Trade Screen (Loading)" width="150"/>

> "Standings" view with values from API loaded

<img src="https://joshzbeck.com/wp-content/uploads/2024/02/Home-Screen-Loading.png" alt="Home View (Loading)" width="150"/> 

> "Home" view (loading with skeleton)

<img src="https://joshzbeck.com/wp-content/uploads/2024/02/Trade-Screen-Loading.png" alt="Trade View (Loading)" width="150"/>

> "Trade" view (loading with skeleton)

#### Additional widget creations

<img src="https://joshzbeck.com/wp-content/uploads/2024/02/New-Trade-Functionality.png" alt="Pending Trades Widget" width="150"/> 

> Adding the "Pending Trades" widget 

#### Splash Screen Updates
*Updated the splash screen with new images, text, and formatting*

<img src="https://joshzbeck.com/wp-content/uploads/2024/02/Updated-Splash-Screen.png" alt="Splash Screen Updates" width="150"/> 

## Challenges

1. **Handling Dart Method Parameters:** Initial challenges arose due to the internal workings of Dart regarding passing properties into methods. Dart passes objects by reference causing unintentional reference chaining due to basic caching being stored in a static `List`.  
>**Solution**: To mitigate this, lists and values were copied before modification, and the cache was nullified upon logout, ensuring separation while reducing API reads.

2. **Data Consistency Issues:** Inconsistencies in stored Airtable data, such as missing players and inconsistently capitalized emails/names, posed challenges. 

> **Solution**: These were addressed within the app by lowercasing values and utilizing optionally chained values to help ensure a more uniform dataset within the app and handle unexpected values.

  
## Summary

By successfully integrating the Airtable backend with the Flutter front-end, implementing 3rd-party authentication mechanisms, addressing data consistency issues, and enhancing user experience, the developer delivered a fully functional fantasy ice hockey app tailored to the non-profit league's needs, providing a streamlined platform for league management and engaging user experience in Flutter ✨🚀
