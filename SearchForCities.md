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

```
