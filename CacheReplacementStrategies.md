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

class LFUCache : public CacheStrategy {
private:
    struct LFUEntry {
        CityData data;
        int frequency;
    };

    list<LFUEntry> cacheList;
    unordered_map<string, list<LFUEntry>::iterator> cacheMap;

public:
    int get(const string& country, const string& city) override {
        string key = makeKey(country, city);
        if (cacheMap.find(key) != cacheMap.end()) {
            cacheMap[key]->frequency++;
            cout << "(From Cache)" << endl;
            return cacheMap[key]->data.population;
        }
        int population = searchCSV(country, city);
        if (population != -1) {
            put(country, city, population);
        }
        return population;
    }

    void put(const string& country, const string& city, int population) override {
        string key = makeKey(country, city);

        if (cacheMap.find(key) != cacheMap.end()) {
            return;
        }

        if (cacheList.size() >= CHACHE_SIZE) {
            auto leastUsed = cacheList.begin();
            for (auto i = cacheList.begin(); i != cacheList.end(); i++) {
                if (i->frequency < leastUsed->frequency || (i->frequency == leastUsed->frequency && distance(cacheList.begin(), i) > distance(cacheList.begin(), leastUsed))) {
                    leastUsed = i;
                }
            }
            cacheMap.erase(makeKey(leastUsed->data.countryCode, leastUsed->data.cityName));
            cacheList.erase(leastUsed);
        }

        cacheList.push_back({{country, city, population}, 1});
        cacheMap[key] = --cacheList.end();
    }
};

class FIFOCache : public CacheStrategy {
private:
    list<CityData> cacheList;
    unordered_map<string, list<CityData>::iterator> cacheMap;

public:
    int get(const string& country, const string& city) override {
        string key = makeKey(country, city);
        if (cacheMap.find(key) != cacheMap.end()) {
            cout << "(From Cache)" << endl;
            return cacheMap[key]->population;
        }
        int population = searchCSV(country, city);
        if (population != -1) {
            put(country, city, population);
        }
        return population;
    }

    void put(const string& country, const string& city, int population) override {
        string key = makeKey(country, city);
        if (cacheMap.find(key) != cacheMap.end()) {
            return;
        }

        if (cacheList.size() >= CHACHE_SIZE) {
            CityData oldest = cacheList.front();
            cacheMap.erase(makeKey(oldest.countryCode, oldest.cityName));
            cacheList.pop_front();
        }

        cacheList.push_back({country, city, population});
        cacheMap[key] = --cacheList.end();
    }
};

class RandomCache : public CacheStrategy {
private:
    vector<CityData> cacheVec;
    unordered_map<string, int> cacheMap;

public:
    int get(const string& country, const string& city) override {
        string key = makeKey(country, city);
        if (cacheMap.find(key) != cacheMap.end()) {
            cout << "(From Cache)" << endl;
            return cacheVec[cacheMap[key]].population;
        }
        int population = searchCSV(country, city);
        if (population != -1) {
            put(country, city, population);
        }
        return population;
    }

    void put(const string& country, const string& city, int population) override {
        string key = makeKey(country, city);
        if (cacheMap.find(key) != cacheMap.end()) {
            return;
        }

        if (cacheVec.size() >= CHACHE_SIZE) {
            int index = rand() % CHACHE_SIZE;
            string oldKey = makeKey(cacheVec[index].countryCode, cacheVec[index].cityName);
            cacheMap.erase(oldKey);
            cacheVec[index] = {country, city, population};
            cacheMap[key] = index;
        }
        else {
            cacheVec.push_back({country, city, population});
            cacheMap[key] = cacheVec.size() - 1;
        }
    }
};

int main() {
    srand(time(0));
    string country;
    string city;
    int choice;

    cout << "Choose cache replacement strategy: " << endl;;
    cout << "1. LFU" << endl;
    cout << "2. FIFO" << endl;
    cout << "3. Random Replacement" << endl;
    cout << "Enter choice (1-3): ";
    cin >> choice;
    cin.ignore();

    CacheStrategy* cache = nullptr;

    switch (choice) {
        case 1:
            cache = new LFUCache();
            break;
        case 2:
            cache = new FIFOCache();
            break;
        case 3:
            cache = new RandomCache();
            break;
        default:
            cout << "Invalid choice." << endl;
            return 1;
    }

    while (true) {
        cout << "Enter a country code (type 'exit' to quit): ";
        getline(cin, country);
        if (country == "exit") {
            break;
        }

        cout << "Enter a city name: ";
        getline(cin, city);

        int pop = cache->get(country, city);
        if (pop != -1) {
            cout << "The population of " << city << ", " << country << " is " << pop << endl;
        }
        else {
            cout << "The city was not found in the file" << endl;
        }
        cout << endl;
    }

    delete cache;

    return 0;
}

```
