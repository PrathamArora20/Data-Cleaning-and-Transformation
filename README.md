# Data-Cleaning-and-Transformation

This project cleans and transforms data using regex

## Execution

To execute the file, change the directory to the main folder 
( in this case "f24-asn1-nDevised" , so its `cd f24-asn1-nDevised` )

Use the following command in the folder directory.

`python3 src/main.py `

## Data

The assignment's raw file Data can be found in the "Data" Folder.

Once the script runs , it will generate 2 new folders. 

1) The clean data can be found in the "clean" folder.

2) The transformed data can be found in the "transformed" folder.

The directory will look something like this

f24-asn1-nDevised/ <br/>
|─ data/<br />
│<br />
|─ clean/<br />
│<br />
|─ transformed/<br />
│ <br />
|─ src/<br />
│  |─ main.py<br />
|<br />
|─ README.md<br />
|─ justification.txt<br />


## Acknowledgment 

We used the regex python library documentation to learn about the "re" library

https://docs.python.org/3/library/re.html


## Justification for design choices.

### Data Cleaning 

So the first data-cleaning decision we made was to read the file by each line instead of all the files at one time. 
We decided this because using regex on each line would be easier and easier for formatting. Breaking the file into individual files gives us more flexibility in using regex and makes it easier.

The first thing that we did was, we went through the raw data file. In the file, we saw that some of the lines were overflowing to the next. Meaning the content was separated into 2 lines. 
This caused us a small hurdle. If we just removed all the lines with '@' and '%' , the lines that overflowed from these headers won't be removed as the extra line doesn't start with a '@' or '%'.
eg:

%mor:	mod|do pro:per|you v|want~inf|to v|give det:poss|her n|baby
	n|bottle ?

We also couldn't just remove all the overflow lines as some character's speech also overflowed to the other line, so we had to retain this information.
So what we did to tackle this problem was, we first checked if the line was an overflow line ( it started with a tab space)
Then looked at the previous line of the overflow line. If the previous line started with "@" or "%", it meant that it was an overflow line from the extraneous line and we could remove it. 
But if it started with a '*' ( as character tags begin with a *), meaning it was a part of character speech, so we kept those lines.

Now with this, All lines began with either '@', '%', or '*', where '@' and '%' were extraneous lines and headers which we removed, and '*' starting for character speech which we keep.

So then we could remove all lines with '@' and '%' . So for that, we removed all lines starting with '@'. What we did was see if the line started with '@', and removed the line.
We also wanted to remove all lines which were not a person's utterance. So we removed all lines starting with '%'

After that, We had the lines with only conversation. But these lines also had the participant's names too. ( Like *CHI:)
So we decided to remove that. We saw the pattern that the participant's name starts with a '*' then there 3 letter initial and ends with a ':'. 
So using this pattern we decided to remove the participants name too. This is because we wanted to keep only the conversation of the people and not the participant tags.

Then we decided to remove all the punctuation marks like '!', '?', '.', '=', '&', '~', '_', and more , so that the words can be easily transformed. 
We also removed all the numbers from the files. This is because there is a lot of punctuation in the text. Some of them are randomly placed in between the texts like '&', 
So we decided to remove those so that it only keeps the texts with meaning in the cleaned folder and later the downstream job for transformation is better as we can get more matches for more words.

Then we moved to brackets. So for brackets, we saw that there was usually content inside the bracket. There were mainly 3 types of brackets 

1. () brackets -> These brackets were usually used to expand a shortened word( eg , (be)c(a)use ). 
So we decided to just remove the brackets and keep the contents of the brackets in the file so the word can be easily found for the downstream job. 
So for the example above if we, removed the bracket and the content inside, it would just become "cuse" which is not found in the dictionary. But if we just remove the brackets, the word becomes "because". and now
in the downstream job for data transformation, the word because can be found.

2. <> brackets -> These brackets had extra content of words. We decided to keep the content but just remove the brackets. 
it as it can also be easily transformed in the downstream jobs. Eg: <what's that> , we just change it to "what's that"

3. [] brackets -> These brackets always contained not important information, which cannot be transformed as mostly it was extraneous information. 
So we decided to remove the brackets AND the content inside.

    3.a. There is an Exception regarding the [] brackets. There are patterns in the file where the word is shortened due to slang or shortened for ease of saying. Because of that, the word is not a proper English word. However, the proper translation of the word is given in [] brackets. 
    eg --> example " what chu [: you] do ? " ==> It deleted chu and removed the [: and ] character to only keep the proper word
    So the sentence now becomes => "what you do ?"
    We do this to improve the understandability of the texts. "chu" will not be transformed properly but replacing it with "you" will help us find a match and transform it.

After that, we decided to remove certain phrases and characters that are extraneous information too as those don't have a valid translation to the English language. 
So we remove phrases like 'xxx', 'you', and ' NAK' as it will also help us match more words in the downstream job especially as these are not important phrases.

There are some word ( with high occurance) that are very close to be found in the CMUdict like "uhoh". "uhoh" exists in the file but in the dictionary, 
there is only "uh-oh". So there were some words we changed so that the dictionary can easily find these words and the downstream job can transform these words easily. 
some of the examples are :
byebye -> bye-bye
uhoh -> uh-oh
uhuh -> uh-huh

We also remove sound words. Sound words are words that mimic a sound made by a person but isnt an actual word. It could be words like 'mm' , 'shh' and more.
So we decided to remove those as well as there is no proper english transformation for those words.

Then we decided to shorten some words to the base form. For that we decided to remove words ending with " 's " and " 'll "from the words. 
This is because for the downstream job where we have to find the pronunciation for the words , the dictionary for that has mostly the base words. 
So we are pruning the word to its base form.

There are some words which we didnt clean more as they are either , 

1. Names of people like SHEM or MIFFY , which cant be transformed as the word does not exist in the library
2. Misspelled words or random words like "PSS" or "MUH" which do not really have an english translation to it. As it does not have an exact english translation to it , 
these words are not possible to transform as these would not exist in a dictionary and would not even map to a english word. Because of that , we decided to leave the 
words as it as without changing it. 


### Data Transformation

So for Data Transformation, We store the dictionary in memory in python making it a dictionary type. 
Then for each word in sentence , we find an appropriate match in the CMUdict and transform the word accordingly.

How the Transformation Works: 
This procedure uses a dictionary to map the words encountered in each text file (the key) to a pronunciation on the cmudict file (the value). 
This is done before the transformation function is ran so that the dictionary is prepared in a hash map, and every word takes O(1) time to access. 
Following this, for every text file in the clean directory, the transformation function is ran. 
The transformation function goes through the file line by line, and splits each line into word. 
Then, each word is mapped to its pronunciation using the dictionary created earlier on. 
If a word not in the dictionary is encountered, then the word is left as is (and not mapped to a translation). 

To further transform the data, for words that were not in the dictionary, the root word was extracted to see if the root exists in the dictionary. 
Common suffixes such as ‘S’, ‘IE’, ‘ING’, ‘Y’, and ‘ED’ were removed from the word, and the root form of the word was checked to see if it existed in the dictionary.
 If the root form existed, then it was used to substitute the sound of the word .
