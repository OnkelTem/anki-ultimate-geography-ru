# How to translate Anki deck

This how-to describes the process of translating Anki deck.

What we gonna do is:

- export deck into JSON;
- extract needed information into a CSV using [jq](https://stedolan.github.io/jq/);
- edit (e.g. translate) this information in Google Spreadsheets;
- export the updated information into JSON array using [Export Sheet Data](https://chrome.google.com/webstore/detail/export-sheet-data/bfdcopkbamihhchdnjghdknibmcnfplk?utm_source=permalink) Google Spreadsheets add-on; 
- apply it as a patch for the original JSON.

## Prerequisites

* Install [Anki for Desktop](https://apps.ankiweb.net/) application if it's not installed yet.

* Install [CrownAnki](https://github.com/Stvad/CrowdAnki) add-on for Anki. It provides additional 
Export/Import options to save/load decks using its own JSON format.

* Install [jq](https://stedolan.github.io/jq/) - a command-line JSON processor.    

## Guide

### Export deck into JSON

Here you have two variants:

- If the deck is distributed in CrowdAnki JSON, then just get its sources.

- If it's a shared deck then install it into Anki for Desktop and export it to CrowdAnki JSON.

In both case you end up with a directory with a json file representing the deck's data and `media/` 
directory with deck's media files.    

## Extract information into CSV

First you need to know the ordinal number of the field you want to update. Open the JSON-file
you got on the previous step and find the `notes` section in there. Here goes a collection of deck's 
notes. Check out the first note and locate the field of your interest. 

For example, let's pick the JSON-file from [Ultimate Geography](https://github.com/axelboc/anki-ultimate-geography) deck:

```json
    "notes": [
        {
            "__type__": "Note", 
            "data": "", 
            "fields": [
                "England", 
                "Constituent country of the United Kingdom.",
                ... 
```

Let's imagine we're translating country names. Notice that they go in the 0th position of the `fields` array. We will 
reference it in our examples below.

Here is how you can export field data into CSV using **jq**:  

```
$ { echo "data"; jq -r '. | .notes[] | .fields[OFFSET] | [.] | @csv' < JSON; } > CSV
```

where:

  - `OFFSET` is the index of the field we're exporting;
  - `JSON` is the original JSON-file;
  - `CSV` is a CSV-file to export data into.

This is how we can export country names:

```
$ { echo "name"; jq -r '. | .notes[] | .fields[0] | [.] | @csv' < Ultimate_Geography.json; } > CountryNames.csv
```

The resulting `CountryNames.csv` is a single-column plain CSV file with a list of countries:

```
$ cat CountryNames.csv | head -n 6
name
"England"
"Scotland"
"United Kingdom"
"Northern Ireland"
"France"
```

Now let's translate them.

## Editing CSV data

You can use any tool to edit your CSV data but since 
[Google Spreadsheets](https://docs.google.com/spreadsheets/u/0/) can export data back into 
JSON it's reasonable to use it. 

Continuing our example, check out [this shared spreadsheet](https://docs.google.com/spreadsheets/d/1b_d4wUQcBL-bLP3WWzYxe7AzkwbJe7NyhKZVQClWx_8/edit?usp=sharing) with Country names.
The first **CountryNames** sheet contains the original data imported from the CSV file.
The second **RU** sheet contains translated country names. Translation is performed
automatically using [*=GoogleTranslate()* function](https://support.google.com/docs/answer/3093331?hl=en).
 
## Export new data into JSON

To export data into JSON we will use [Export Sheet Data](https://chrome.google.com/webstore/detail/export-sheet-data/bfdcopkbamihhchdnjghdknibmcnfplk?utm_source=permalink) 
Google Spreadsheets add-on. 

Switch to the appropriate sheet and open Export Sheet Data's sidebar:

**Add-ons > Export Sheet Data > Open Sidebar**

Ensure that `Select Format` field is set to `"JSON"` and tick the following checkboxes:

- `[x] Export value arrays` in the `JSON group`
- `[x] Export contents as array` in the `Advanced JSON` group 

Click **Export** button to download the JSON data or **Visualize** button to preview it first.

The resulting JSON-file should be just a plain array:

```
$ cat RU.json | head -n 6
[
  "Англия",
  "Шотландия",
  "Великобритания",
  "Северная Ирландия",
  "Франция",
```

## Patching the original JSON-file

To apply the patch we will use **jq** one more time. In a generic form our command look like:  

```
$ jq -s '
         .[1] as $array
           |
         .[0]
           |
         .notes 
           |= reduce range(0;length) as $i (.;
               .[$i].fields[OFFSET] = $array[$i])' \
    JSON PATCH > PATCHED
```

where:
 
- `OFFSET` is the index of the field we're updating;
- `JSON` is the original JSON-file;
- `PATCH` is our patch file;
- `PATCHED` is a patched version of the original JSON-file.

Back to our example, this is real command which applies countries translation to 
[Ultimate Geography](https://github.com/axelboc/anki-ultimate-geography) deck: 

```
$ jq --indent 4 -s '
         .[1] as $array
           |
         .[0]
           |
         .notes 
           |= reduce range(0;length) as $i (.;
               .[$i].fields[0] = $array[$i])' \
    Ultimate_Geography.json RU.json | sed 's/,$/, /' > Ultimate_Geography_ru.json  
```

Here `-- indent 4` and `sed 's/,$/, /'` augmentations make the final JSON look 
almost the same as the original one which simplifies real patching.  




