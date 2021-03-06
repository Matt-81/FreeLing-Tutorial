```C++
#include <iostream>
#include "freeling.h"
using namespace std;

int main(int argc, char* argv[]){

  // set locale to an UTF8 compatible locale
  freeling::util::init_locale(L"default");

  // read FreeLing installation path if given, use default otherwise
  wstring ipath = L"/usr/local";
  if (argc > 1) ipath = freeling::util::string2wstring(argv[1]);

  // path to data common to all languages
  wstring cpath = ipath+L"/share/freeling/common/";

  // creates a language identifier with the default config file
  freeling::lang_ident di(cpath+L"lang_ident/ident.dat");
  
  set<wstring> candidates=set<wstring>(); // list of languages to consider. 
                                          // Empty -> all known languages
  vector<pair<double,wstring> > result;
  wstring line;
  while (getline(wcin,line)) {
    wcout << L"-----------------------------------" << endl;
    wcout << L"Input text: [" << line << L"]" << endl;

    // classificate the read text line and obtain a sorted 'result' vector <language_code, perplexity>
    di.rank_languages(result, line, candidates);

    // The funcion identify_language will return the code for the best language, 
    // or "none" if no language model yields a small enough perplexity (according
    // to the threshold set in that language model)
    wstring best_l = di.identify_language (line, candidates);
    wcout << L"Best language: "<<best_l<<endl;

    // You can also output get a sorted list of increasing perplexity,
    // in case you want to take the decision yourself.
    wcout<<L"Increasing perplexity list:"<<endl;
    vector<pair<double,wstring> >::iterator i;
    for (i=result.begin(); i!=result.end(); i++)
      wcout<<L"  "<<i->second<<L" "<<i->first<<endl;     
  }
}



```
