```cpp

#include <iostream>
#include <fstream>
#include <sstream>
#include <list>
#include <unordered_map>

using namespace std;

const int CHACHE_SIZE = 10;
const string CSV_FILE = "world_cities.csv";

struct CityData {
    string countryCode;
    string cityName;
    int population;
};

string makeKey(const string& country, const string& city) {
    return country + "," + city;
}

class CacheStrategy {
public:
    virtual int get(const string& country, const string& city) = 0;
    virtual void put(const string& country, const string& city, int population) = 0;
    virtual ~CacheStrategy() {}
};

int searchCSV(const string& country, const string& city) {
    ifstream file(CSV_FILE);
    if (!file.is_open()) {
        cerr << "Error: Could not open file " << CSV_FILE << endl;
        return -1;
    }

    string line;
    getline(file, line); //skip the header

    while (getline(file, line)) {
        stringstream ss(line);
        string token;
        string cCode;
        string cName;
        int pop;

        getline(ss, cCode, ',');
        getline(ss, cName, ',');
        getline(ss, token, ',');
        pop = stoi(token);

        if (cCode == country && cName == city) {
            return pop;
        }
    }

    return -1;
}

int main() {
    string country;
    string city;

    while (true) {
        cout << "Enter a country code (type 'exit' to quit): ";
        //cin >> country;
        getline(cin, country);
        if (country == "exit") {
            break;
        }

        cout << "Enter a city name: ";
        //cin.ignore();
        getline(cin, city);

        int pop = getPopulation(country, city);
        if (pop != -1) {
            cout << "The population of " << city << ", " << country << " is " << pop << endl;
        }
        else {
            cout << "The city was not found in the file" << endl;
        }
        cout << endl;
    }

    return 0;
}

```
