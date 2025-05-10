```cpp

#include <iostream>
#include <unordered_map>
#include <sstream>
#include <fstream>
#include <vector>
#include <list>

using namespace std;

const int CACHE_SIZE = 10;
const string CSV_FILE = "world_cities.csv";

struct TrieNode {
    bool isEndOfWord;
    unordered_map<char, TrieNode*> children;
    unordered_map<string, int> countryPopMap;

    TrieNode() : isEndOfWord(false) {}
};

class CityTrie {
private:
    TrieNode* root;

public:
    CityTrie() {
        root = new TrieNode();
    }

    void insert(const string& city, const string& country, int population) {
        TrieNode* curr = root;
        for (int c : city) {
            c = tolower(c);
            if (curr->children.count(c)==0) {
                curr->children[c] = new TrieNode();
            }
            curr = curr->children[c];
        }
        curr->isEndOfWord = true;
        curr->countryPopMap[country] = population;
    }

    int getPopulation(const string& city, const string& country) {
        TrieNode* curr = root;
        for (int c : city) {
            c = tolower(c);
            if (curr->children.count(c)==0) {
                return -1;
            }
            curr = curr->children[c];
        }
        if (curr->isEndOfWord && curr->countryPopMap.count(country)) {
            return curr->countryPopMap[country];
        }
        return -1;
    }
};

struct CityData {
    string countryCode;
    string cityName;
    int population;
};

string makeKey(const string& country, const string& city) {
    return country + "," + city;
}

void loadCSV(CityTrie& cityTrie, string filename) {
    ifstream file(filename);
    if (!file.is_open()) {
        cerr << "Error: Could not open file " << filename << endl;
        return;
    }

    string line;
    getline(file, line);

    while (getline(file, line)) {
        stringstream ss(line);
        string country;
        string city;
        string popstr;

        int population;

        getline(ss, country, ',');
        getline(ss, city, ',');
        getline(ss, popstr, ',');

        population = stoi(popstr);

        cityTrie.insert(city, country, population);
    }

    file.close();
}

class CacheStrategy {
public:
    virtual int get(const string& country, const string& city) = 0;
    virtual void put(const string& country, const string& city, int population) = 0;
    virtual ~CacheStrategy() {}
};

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
        return -1;
    }

    void put(const string& country, const string& city, int population) override {
        string key = makeKey(country, city);

        if (cacheMap.find(key) != cacheMap.end()) {
            return;
        }

        if (cacheList.size() >= CACHE_SIZE) {
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
        return -1;
    }

    void put(const string& country, const string& city, int population) override {
        string key = makeKey(country, city);
        if (cacheMap.find(key) != cacheMap.end()) {
            return;
        }

        if (cacheList.size() >= CACHE_SIZE) {
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
        return -1;
    }

    void put(const string& country, const string& city, int population) override {
        string key = makeKey(country, city);
        if (cacheMap.find(key) != cacheMap.end()) {
            return;
        }

        if (cacheVec.size() >= CACHE_SIZE) {
            int index = rand() % CACHE_SIZE;
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

    CityTrie cityTrie;
    loadCSV(cityTrie, CSV_FILE);

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

    string country;
    string city;

    while (true) {
        cout << "Enter a country code (type 'exit' to quit): ";
        getline(cin, country);
        if (country == "exit") {
            break;
        }

        cout << "Enter a city name: ";
        getline(cin, city);

        int pop = cache->get(country, city);
        if (pop == -1) {
            pop = cityTrie.getPopulation(city, country);
            if (pop != -1)
                cache->put(country, city, pop);
        }

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
