#include<bits/stdc++.h>

using namespace std;

ifstream in;

struct FrequentItemsets
{
  vector<vector<string>> itemsets;
  vector<int> supports;
};

struct AssociationRule
{
  vector<string> lhs;
  vector<string> rhs;
};

struct Association
{
  vector<AssociationRule> rules;
  vector<int> supports;
  vector<double> confidences;
};


bool read_transactions(string filename, vector<string>& transaction)
{
  string line;
  transaction.clear();
  int start, end;

  while (true)
  {
    if (!getline(in , line))
      return false;
    start = line.find_first_of('[');
    end = line.find_first_of(']');
    if (start == line.npos || end == line.npos)
      continue;
    break;
  }

  istringstream iss(line.substr(start + 1, end - start - 1));
  string it;
  while (getline(iss, it, ','))
    transaction.push_back(it);
  return true;
}

void Reset()
{
  in.clear();
  in.seekg(0);
}


FrequentItemsets Frequent_one_itemsets(string filename, vector <string> transaction, int minSup)
{
  unordered_map <string, int> map;
  Reset();
  while (read_transactions(filename, transaction)) {
    for (auto &item : transaction) {
      map[item]++;
    }
  }

  FrequentItemsets result;
  for (auto & pair : map)
    if (pair.second >= minSup)
    {
      result.itemsets.push_back(vector <string> (1, pair.first));
      result.supports.push_back(pair.second);
    }

  return result;
}

FrequentItemsets Merge(FrequentItemsets & It, const FrequentItemsets & result)
{
  for (auto & itemset : result.itemsets)
    It.itemsets.push_back(itemset);

  for (auto support : result.supports)
    It.supports.push_back(support);

  return It;
}

bool IsJoinItemsets(const vector <string> & itemset1, const vector <string> & itemset2)
{
  int j = itemset1.size();
  for (int i = 0; i < j - 1; ++i)
    if (itemset1[i] != itemset2[i])
      return false;
  return itemset1[j - 1] < itemset2[j - 1];

}

vector<string> JoinItemsets(const vector <string>& itemset1, const vector <string>& itemset2)
{
  vector<string> NewItemset = itemset1;
  NewItemset.push_back(itemset2.back());
  return NewItemset;
}

bool hasInfrequentSubset(const vector <string>& candidate, const vector <vector<string>> & freqItemsets)
{
  if (candidate.size() == 2)
    return false;

  for (int i = 0; i < candidate.size(); i++)
  {
    vector <string> candidateSubset;
    for (int j = 0; j < candidate.size(); j++)
      if (j != i)
        candidateSubset.push_back(candidate[j]);

    auto pred = [&](const vector <string>& itemset)
    {
      return candidateSubset == itemset;
    };

    if (find_if(freqItemsets.begin(), freqItemsets.end(), pred) == freqItemsets.end())
      return true;
  }

  return false;
}


vector<vector<string>> genCandidates(const vector <vector<string>>& freqItemsets, int minSup)
{
  vector<vector<string>> candidates;
  for (auto &itemset1 : freqItemsets)
    for (auto &itemset2 : freqItemsets)
      if (IsJoinItemsets(itemset1, itemset2))
      {
        auto candidate = JoinItemsets(itemset1, itemset2);
        if (!hasInfrequentSubset(candidate, freqItemsets))
          candidates.push_back(candidate);
      }

  return candidates;
}

FrequentItemsets pruneCandidates(const vector<vector<string>>& candidates, string filename, vector<string> transaction, int minSup)
{
  FrequentItemsets result;
  vector <int> ctr(candidates.size(), 0);
  Reset();
  while (read_transactions(filename, transaction))
  {
    sort(transaction.begin(), transaction.end());
    for (int i = 0; i < candidates.size(); ++i)
    {
      const vector <string> & candidate = candidates[i];
      if (includes(transaction.begin(), transaction.end(), candidate.begin(), candidate.end()))
        ctr[i]++;
    }

  }

  for (int i = 0; i < candidates.size(); ++i)
    if (ctr[i] >= minSup) {
      result.itemsets.push_back(candidates[i]);
      result.supports.push_back(ctr[i]);
    }

  return result;
}


FrequentItemsets AprioriAlgorithm(string filename, vector < string > transaction, int minSup)
{
  //int i = 0;
  FrequentItemsets result;
  auto FrequentItemsets = Frequent_one_itemsets(filename, transaction, minSup);
  while (!FrequentItemsets.itemsets.empty())
  {
    result = Merge(result, FrequentItemsets);
    auto candidates = genCandidates(FrequentItemsets.itemsets, minSup);
    FrequentItemsets = pruneCandidates(candidates, filename, transaction, minSup);
  }

  return result;
}

void LoadDatasetFromFile(string filename)
{
  in .open(filename, fstream::in);

  if (!in ) {
    cout << "Fail to load file." << endl;
    exit(1);
  }
}




void ConstructRule(vector<string> &lhs, const vector<string> &itemset, int itemsetSupport, int depth, const map<vector<string>, int> &SupMap, double minConf, Association &result)
{
  if (depth == itemset.size())
  {
    if (lhs.empty() || lhs.size() == itemset.size())
      return;
    double conf = (double)itemsetSupport / SupMap.at(lhs);
    if (conf >= minConf)
    {
      AssociationRule rule;
      rule.lhs = lhs;
      set_difference(itemset.begin(), itemset.end(), lhs.begin(), lhs.end(), back_inserter(rule.rhs));

      result.rules.push_back(rule);
      result.supports.push_back(itemsetSupport);
      result.confidences.push_back(conf);
    }
    return;
  }

  ConstructRule(lhs, itemset, itemsetSupport, depth + 1, SupMap, minConf, result);
  lhs.push_back(itemset[depth]);
  ConstructRule(lhs, itemset, itemsetSupport, depth + 1, SupMap, minConf, result);
  lhs.pop_back();

}

void ConstructRule(const vector<string> &itemset, const map<vector<string>, int> &SupMap, double minConf, Association &result)
{
  if (itemset.size() <= 1)
    return;
  vector<string> lhs;
  ConstructRule(lhs, itemset, SupMap.at(itemset), 0, SupMap, minConf, result);
}





Association genAssociationRules(FrequentItemsets &result, double minConfidence)
{
  map<vector<string>, int> SupMap;
  for (size_t i = 0; i < result.itemsets.size(); ++i)
    SupMap.insert(make_pair(result.itemsets[i], result.supports[i]));

  Association assocResult;
  for (auto &itemset : result.itemsets)
    ConstructRule(itemset, SupMap, minConfidence, assocResult);

  return assocResult;
}




int main() {


  vector < string > transaction;

  cout << "Enter dataset file name:\n";
  string dataset_name ; //dataset_added[].txt
  cin >> dataset_name;


  cout << "Enter minimum support\n";
  float minimum_support;
  cin >> minimum_support;



  cout << "Enter minimum confidence\n";
  double minConf;
  cin >> minConf;

  LoadDatasetFromFile(dataset_name);

  cout << "\n";


  cout << "*******************************************************************************" << endl;
  FrequentItemsets aprioriOutput = AprioriAlgorithm(dataset_name, transaction, minimum_support);
  cout << "===============================================================================" << endl;

  cout << "The size of Frequent itemsets: " << aprioriOutput.itemsets.size() << "\n";
  cout << "===============================================================================" << endl;

  for (int i = 0; i < (int) aprioriOutput.itemsets.size(); ++i) {
    int m = aprioriOutput.itemsets[i].size();

    for (int j = 0; j < m; ++j) {
      if (j == (m - 1)) cout << aprioriOutput.itemsets[i][j];
      else cout << aprioriOutput.itemsets[i][j] << ", ";
    }

    cout << ": " << aprioriOutput.supports[i] << "\n";
  }



  Association AssociationOutput = genAssociationRules(aprioriOutput, minConf);

  cout << "\n";

  cout << "===============================================================================" << endl;
  cout << "The size of Association rules: " << AssociationOutput.rules.size() << "\n";
  cout << "===============================================================================" << endl;



  for (int i = 0; i < (int)AssociationOutput.rules.size(); ++i) {

    int size_lhs = AssociationOutput.rules[i].lhs.size();
    int size_rhs = AssociationOutput.rules[i].rhs.size();

    //printing lhs
    for (int k = 0; k < size_lhs; k++) {
      cout << AssociationOutput.rules[i].lhs[k] << "\t";

    }

    cout << "-->";

    //printing rhs
    for (int k = 0; k < size_rhs; k++) {
      cout << "\t" << AssociationOutput.rules[i].rhs[k] << "\t";

    }

    cout << "{ Support: " << AssociationOutput.supports[i] << " \t confidences: " << AssociationOutput.confidences[i] << "}";
    cout << endl;

  }






  in.close();
}