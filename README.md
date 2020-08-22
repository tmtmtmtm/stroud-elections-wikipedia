
The code and queries etc here are unlikely to be updated as my process
evolves. Later repos will likely have progressively different approaches
and more elaborate tooling, as my habit is to try to improve at least
one part of the process each time around.

---------

Step 1: Configure config.json
=============================

All the relevant metadata now lives in config.json: ideally nothing will
need tweaked after this. We need to be careful here to get the history
of Wikidata IDs for the constituency correct.

Step 1: Scrape the results
==========================

```sh
bundle exec ruby scraper.rb config.json | tee wikipedia.csv
```

Step 2: Check for missing party IDs
===================================

```sh
xsv search -v -s party 'Q' wikipedia.csv
```

`wb ce create-new-party.js` was too lagged, so I created these manually:

* Free Public Transport -> Q98555021
* Powell Conservative -> Q98555022


Ste 3: Check for missing election IDs
=====================================

```sh
xsv search -v -s election 'Q' wikipedia.csv | xsv select electionLabel | uniq
```

* "Stroud by-election, 12 February 1875" -> Q98555025
* "Stroud by-election, 27 July 1874" -> Q98555026
* "Stroud by-election, 18 May 1874" -> Q98555030
* "By-election, 20 August 1867" -> Q98555029


Step 4: Generate possible missing person IDs
============================================

```sh
xsv search -v -s id 'Q' wikipedia.csv | xsv select name | tail +2 |
  sed -e 's/^/"/' -e 's/$/"@en/' | paste -s - |
  xargs -0 wd sparql find-candidates.js |
  jq -r '.[] | [.name, .item.value, .election.label, .constituency.label, .party.label] | @csv' |
  tee candidates.csv
```

Step 5: Combine Those
=====================

```sh
xsv join -n --left 2 wikipedia.csv 1 candidates.csv | xsv select '10,1-8' | sed $'1i\\\nfoundid' | tee combo.csv
```

Step 6: Generate QuickStatements commands
=========================================

```sh
bundle exec ruby generate-qs.rb config.json | tee commands.qs
```

Then sent to QuickStatements as https://editgroups.toolforge.org/b/QSv2T/1598092296013/
