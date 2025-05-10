```cpp

#include <iostream>
#include <unordered_map>
#include <sstream>
#include <fstream>
#include <vector>
#include <list>
#include <random>
#include <string>
#include <chrono>

using namespace std;
using namespace chrono;

const int CACHE_SIZE = 10;
const int NUM_LOOKUPS = 1000;
const string CSV_FILE = "world_cities.csv";
const string TEST_OUTPUT = "results.csv";

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
            //cout << "(From Cache)" << endl;
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
            //cout << "(From Cache)" << endl;
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
            //cout << "(From Cache)" << endl;
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

vector<pair<string, string>> generateCities(CityTrie& trie, int numCities) {
    vector<pair<string, string>> cities;
    ifstream file(CSV_FILE);
    string line;
    getline(file, line);

    vector<pair<string, string>> cityList;
    while (getline(file, line)) {
        stringstream ss(line);
        string country;
        string city;
        getline(ss, country, ',');
        getline(ss, city, ',');
        cityList.push_back({city, country});
    }

    random_device rd;
    mt19937 gen(rd());
    uniform_int_distribution<> dist(0, cityList.size() - 1);

    for (int i = 0; i < numCities; i++) {
        cities.push_back(cityList[dist(gen)]);
    }

    return cities;
}

void runTest(const string& strategyName, CacheStrategy& cache, CityTrie& trie, const vector<pair<string, string>>& cities) {
    int cacheHits = 0;
    auto start = high_resolution_clock::now();

    for (const auto& city : cities) {
        int pop = cache.get(city.second, city.first);
        if (pop == -1) {
            pop = trie.getPopulation(city.first, city.second);
            if (pop != -1) {
                cache.put(city.second, city.first, pop);
            }
        }
        else {
            cacheHits++;
        }
    }

    auto end = high_resolution_clock::now();
    double duration = duration_cast<microseconds>(end - start).count();
    double hitRate = (cacheHits * 100.0) / cities.size();

    ofstream logFile(TEST_OUTPUT, ios::app);
    logFile << strategyName << "," << duration << " ms," << duration / cities.size() << " ms," << hitRate << "%" << endl;
    logFile.close();
}

int main() {
    CityTrie cityTrie;
    loadCSV(cityTrie, CSV_FILE);
    vector<pair<string, string>> rdmCities = generateCities(cityTrie, NUM_LOOKUPS);

    ofstream logFile(TEST_OUTPUT, ios::app);
    logFile << "Strategy,Total Time,Average Time,Cache Hit Rate" << endl;
    logFile.close();

    LFUCache lfuCache;
    runTest("LFU", lfuCache, cityTrie, rdmCities);

    FIFOCache fifoCache;
    runTest("FIFO", fifoCache, cityTrie, rdmCities);

    RandomCache randomCache;
    runTest("Random", randomCache, cityTrie, rdmCities);

    cout << "Load testing completed. Results saved in " << TEST_OUTPUT << endl;

    return 0;
}

```
