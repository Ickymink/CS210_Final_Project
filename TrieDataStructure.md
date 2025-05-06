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

```
