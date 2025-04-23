```cpp

#include <iostream>
#include <fstream>
#include <sstream>
#include <list>
#include <unordered_map>

using namespace std;

struct CityData {
    string countryCode;
    string cityName;
    int population;
};

list<CityData> cacheList;
unordered_map<string, list<CityData>::iterator> cacheMap;

const int CHACHE_SIZE = 10;
const string CSV_FILE = "world_cities.csv";

string makeKey(const string& country, const string& city) {
    return country + "," + city;
}

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

void updateCache(const string& country, const string& city, int population) {
    string key = makeKey(country, city);

    if (cacheMap.find(key) != cacheMap.end()) {
        cacheList.erase(cacheMap[key]);
        cacheMap.erase(key);
    }

    cacheList.push_front({key, country, population});
    cacheMap[key] = cacheList.begin();

    if (cacheList.size() > CHACHE_SIZE) {
        CityData last = cacheList.back();
        cacheMap.erase(makeKey(last.countryCode, last.cityName));
        cacheList.pop_back();
    }
}

int getPopulation(const string& country, const string& city) {
    string key = makeKey(country, city);

    if (cacheMap.find(key) != cacheMap.end()) {
        cout << "(From Cache)"; //to signal that the data was recieved from cache
        return cacheMap[key]->population;
    }

    int population = searchCSV(country, city);
    if (population != -1) {
        updateCache(country, city, population);
    }
    return population;
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
