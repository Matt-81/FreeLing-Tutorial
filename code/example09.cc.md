```C++
#include <iostream>
#include "freeling.h"
#include "freeling/morfo/analyzer.h"
using namespace std;


///////////////////////////////////////////////////
// Load an ad-hoc set of configuration options

freeling::analyzer::config_options fill_config(const wstring &lang, const wstring &ipath) {

  freeling::analyzer::config_options cfg;

  // Language of text to process
  cfg.Lang = lang;
 
  // path to language specific data
  wstring lpath = ipath + L"/share/freeling/" + cfg.Lang + L"/";

  // Tokenizer configuration file
  cfg.TOK_TokenizerFile = lpath + L"tokenizer.dat";
  // Splitter configuration file
  cfg.SPLIT_SplitterFile = lpath + L"splitter.dat";
  // Morphological analyzer options
  cfg.MACO_Decimal = L".";
  cfg.MACO_Thousand = L",";
  cfg.MACO_LocutionsFile = lpath + L"locucions.dat";
  cfg.MACO_QuantitiesFile = lpath + L"quantities.dat";
  cfg.MACO_AffixFile = lpath + L"afixos.dat";
  cfg.MACO_ProbabilityFile = lpath + L"probabilitats.dat";
  cfg.MACO_DictionaryFile = lpath + L"dicc.src";
  cfg.MACO_NPDataFile = lpath + L"np.dat";
  cfg.MACO_PunctuationFile = lpath + L"../common/punct.dat";
  cfg.MACO_ProbabilityThreshold = 0.001;

  // Sense annotator and WSD config files
  cfg.SENSE_ConfigFile = lpath + L"senses.dat";
  cfg.UKB_ConfigFile = lpath + L"ukb.dat";
  // Tagger options
  cfg.TAGGER_HMMFile = lpath + L"tagger.dat";
  cfg.TAGGER_ForceSelect = freeling::RETOK;
  // Statistical dependency parser & SRL config file
  cfg.DEP_TreelerFile = lpath + L"dep_treeler/dependences.dat";   

  // NEC config file. This module will not be loaded
  cfg.NEC_NECFile = L"";
  // Chart parser config file. This module will not be loaded
  cfg.PARSER_GrammarFile = L"";
  // Rule based dependency parser config files. This module will not be loaded
  cfg.DEP_TxalaFile = L"";
  // Coreference resolution config file. This module will not be loaded
  cfg.COREF_CorefFile = L"";

  return cfg;
}


///////////////////////////////////////////////////
// Load an ad-hoc set of invoke options

freeling::analyzer::invoke_options fill_invoke() {

  freeling::analyzer::invoke_options ivk;

  // Level of analysis in input and output
  ivk.InputLevel = freeling::TEXT;
  // We can not request higher analysis levels (e.g. coreference) because 
  // we didn't load the needed modules.
  // But we can use this option to lowe the analysis level at will during
  // our application execution.
  ivk.OutputLevel = freeling::DEP; 
  
  // activate/deactivate morphological analyzer modules
  ivk.MACO_UserMap = false;
  ivk.MACO_AffixAnalysis = true;
  ivk.MACO_MultiwordsDetection = true;
  ivk.MACO_NumbersDetection = true;
  ivk.MACO_PunctuationDetection = true;
  ivk.MACO_DatesDetection = true;
  ivk.MACO_QuantitiesDetection  = true;
  ivk.MACO_DictionarySearch = true;
  ivk.MACO_ProbabilityAssignment = true;
  ivk.MACO_CompoundAnalysis = false;
  ivk.MACO_NERecognition = true;
  ivk.MACO_RetokContractions = false;
    
  ivk.SENSE_WSD_which = freeling::UKB;
  ivk.TAGGER_which = freeling::HMM;

  // since we only created dep_treeler parser, we can not set the parser to use to another one.
  // If we had loaded both parsers, we could change the used parsed at will with this option
  ivk.DEP_which = freeling::TREELER;

  // since we did not load the module, setting this to true would trigger an error.
  // if the module was created, we could activate/deactivate it at will with this option.
  ivk.NEC_NEClassification = false; 

  return ivk;
}


//---------------------------------------------
// Extract lemma and sense of word 'w' and store them
// in 'lem' and 'sens' respectively
//---------------------------------------------

void extract_lemma_and_sense(const freeling::word &w, wstring &lem, wstring &sens) {
   lem = w.get_lemma();
   if (not w.get_senses().empty()) sens = w.get_senses().begin()->first;
   else sens=L"";
}

//---------------------------------------------
// Do whatever is needed with analyzed sentences
//---------------------------------------------

void ProcessSentences(const list<freeling::sentence> &ls) {

  // for each sentence in list
  for (list<freeling::sentence>::const_iterator s=ls.begin(); s!=ls.end(); ++s) {

    // for each predicate in sentence
    for (freeling::sentence::predicates::const_iterator pred=s->get_predicates().begin(); pred!=s->get_predicates().end(); ++pred) { 

      wstring lsubj=L"", ssubj=L"", ldobj=L"", sdobj=L"";
      // for each argument of the predicate
      for (freeling::predicate::const_iterator arg=pred->begin(); arg!=pred->end(); ++arg) {

         // if the argument is A1, store lemma and synset in ldobj, sdobj
         if (arg->get_role()==L"A1") {
           extract_lemma_and_sense((*s)[arg->get_position()], ldobj, sdobj);
         }

         // if the argument is A0, store lemma and synset in lsubj, ssubj
         else if (arg->get_role()==L"A0") {
           extract_lemma_and_sense((*s)[arg->get_position()], lsubj, ssubj);

           // Get tree node corresponding to the word marked as argument head:
           freeling::dep_tree::const_iterator head = s->get_dep_tree().get_node_by_pos(arg->get_position());
           // check if the node has dependency is "by" in passive structure
           if (lsubj == L"by" and head->get_label() == L"LGS") {
             // advance to its child (it is a preorder iterator, so advancing 
             // one step moves us to the child)
             ++head;
             // use child as actual subject
             extract_lemma_and_sense(head->get_word(), lsubj, ssubj);
           }
         }
      }
      
      // if the predicate had both A0 and A1, we found a complete SVO triple. Let's output it.
      if (lsubj!=L"" and ldobj!=L"") {
        // extract lema and sense for the predicate
        wstring lpred, spred;
        extract_lemma_and_sense((*s)[pred->get_position()], lpred, spred);

        wcout << L"SVO : (pred:    " << lpred << L" [" << spred << L"]" << endl;
        wcout << L"       subject: " << lsubj << L" [" << ssubj << L"]" << endl; 
        wcout << L"       dobject: " << ldobj << L" [" << sdobj << L"]" << endl;
        wcout << L"      )" << endl;
      }      
    }
  }
}


/////////////   MAIN PROGRAM  /////////////////////

int main (int argc, char **argv) {

  // set locale to an UTF8 compatible locale 
  freeling::util::init_locale(L"default");

  // get requested language from arg1, or English if not provided
  wstring lang = L"en";
  if (argc > 1) lang = freeling::util::string2wstring(argv[1]);
  // get installation path to use from arg2, or use /usr/local if not provided
  wstring ipath = L"/usr/local";
  if (argc > 2) ipath = freeling::util::string2wstring(argv[2]);

  // set config options (which modules to create, with which configuration)
  freeling::analyzer::config_options cfg = fill_config(lang, ipath);
  // create analyzer
  freeling::analyzer anlz(cfg);

  // set invoke options (which modules to use. Can be changed in run time)
  freeling::analyzer::invoke_options ivk = fill_invoke();
  // load invoke options into analyzer
  anlz.set_current_invoke_options(ivk);

  // load document to analyze
  wstring text;  
  wstring line;
  while (getline(wcin,line)) 
    text = text + line + L"\n";

  // analyze text, leave result in ls
  list<freeling::sentence> ls;
  anlz.analyze(text,ls);

  ProcessSentences(ls);
}

```
