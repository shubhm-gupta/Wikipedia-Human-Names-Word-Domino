# Word Domino of Human Names Extracted from Wikidpedia Dumps

This is a fun exercise done  with [wikipedia dumps](https://dumps.wikimedia.org/enwiki/latest/)
It is consists of two important parts:

A) Extracting the human names from the Wikipedia Dumps  
B) Creating the Word Domino from the extracted names

A simple domino looks like this:

![image](https://github.com/shubhm-gupta/Wikipedia-Human-Names-Word-Domino/blob/master/domino.jpg)

## A. Extracting the names from the Wikipedia Dumps

This is done on a Hadoop Cluster. If you have access to any Hadoop Cluster or have your own single node cluster, then follow the next steps:

* Download the dump   
    ``` wget https://dumps.wikimedia.org/enwiki/20181120/enwiki-20181120-pages-articles- multistream.xml.bz2 & ```    
    (This link expires once new dumps are uploaded. So make sure to use the latest wikipedia bzip dump)

* Unzip the file   
    ``` bzip2 -dk enwiki-20181120-pages-articles-multistream.xml.bz2```

* Upload the dump file to HDFS    
  ```hadoop fs -put enwiki-20181120-pages-articles-multistream.xml <HDFS PATH>```
  
#### How to extract only human names from the dump that contains everything?

We  started  with  the  analysis  of  the  Wikipedia  dump  file  and  tried  to  identify  the information that could help us shortlisting the titles with “Person names”. Initially, we looked up for  famous  personalities  like  Bill  Gates  and  tried  to  understand  the  xml  dump  of  a  page.  We noticed that the page contained “Birth”, “Death”, “was born on” tags and text in the dump file. Our next step was to find look up for lesser known personalities and verify the same information. However,  we  noticed  that  this  time  pages  were  too  generic  with  very  little  information  and majorly  missing  these  tags.  Then  we  tried  various  other  text  grep  on  the  XML  dump  like bibliography and few other filters, but we could find any uniformity of information across all the pages. Finally, we looked up the page category and Bingo!

We found the tag that we were looking for. All the pages with human titles had a generic tag “**bio-stub**” and other specific pages had “US-bio-stub”, “Sports-bio-stub” and many more.  But the  bio-stub  tag  was  generic  for  all  human  titled  pages.  So  we  used  it  for  extracting  titles  of pages with human names as discussed in the next section.

This helped us to get all the names of the persons along with some junk data which we removed with the help of the sed command in local file system.

### How to run this on Apache Hadoop and Apache Pig?

* Logged into Apache Pig using pig command.  
```$ pig  ```
    
* Loaded the file to variable A using XMLLoader and splitting the XML by pages.  
```A    =    LOAD    '/user/darora2/Project/enwiki-20181120-pages-articles-multistream.xml' using    org.apache.pig.piggybank.storage.XMLLoader('page') as (x:chararray);  ```
    
* Applied filter on the data with the help of XPath if text tag contains bio-stub in it.  
```B = FOREACH A GENERATE XPath(x,'page/revision/text[contains(text(),"bio- stub")]/../../title') ; ``` 
* Stored the names into Local File System  
```STORE B into '/user/darora2/Project/Names' using PigStorage(';');   ```
* ```$ quit  ```
* Merged the 520 files generated having names to one file.  
```$ hadoop fs -getmerge  /user/darora2/Project/Names /users/darora2/Project/Name.txt   ```
* Removed empty lines having spaces using the following command:   
```$ awk 'NF' Name.txt > Names.txt   ```
* Checking the number of names.  
```$ wc -l Names.txt   ```
* Removed unnecessary data using the below commands: 
```
    sed "/Category/d" Names.txt > FilteredName1.txt  
    sed "/Wikipedia/d" FilteredName1.txt > FilteredName2.txt  
    sed "/Template/d" FilteredName2.txt > FilteredName3.txt  
    sed "/Portal/d" FilteredName3.txt > FilteredName4.txt  
    sed "/File/d" FilteredName4.txt > FilteredName5.txt  
    sed -e "s/([^()]*)//g" FilteredName5.txt > FilteredName6.txt  
    sed -e "s/Jr.//g" FilteredName6.txt > FilteredName7.txt  
    sed -e "s/Sr.//g" FilteredName7.txt > FilteredName8.txt
    sed -e "s/,.*$//" FilteredName8.txt > FilteredName9.txt  
    sed -e "s/Draft://g" FilteredName9.txt > FilteredName10.txt  
    sed "/[0-9]/d" FilteredName10.txt > FilteredName11.txt  
```
* $ ``` cp FilteredName11.txt People_Names_Final.txt   ```  
* $ ```  wc -l People_Names_Final.txt  ```  
* Used WinSCP to download the People_Names_Final.txt file to local desktop.

## B. Create Word Domino

We can create a word domino in python. First, we have to create a board and dominoes. Then we read  all  the  names  from  the  text  file  and  only  use  2500  randomly  sampled  names  out  of  the whole list. Then we need to find synonyms of each name to match them with similar names. The final  result  is  the  visualization  of  the  board  with  dominoes  placed  according  to  other  slightly matching first names or last names.  We divided the names between the first name and last name and evaluated using a key-value pair. Doing this as a local script takes a heavy amount of load and time and its unfeasible to do it on all data. So we used Hadoop MapReduce to find the names which are similar and then we generated the visualization using python. 

```
$ module load tensorflow/1.10-anaconda3 

$ hadoop jar hadoop-streaming.jar  -file mapper.py -mapper mapper.py  -file reducer.py -reducer reducer.py  -input People_Names_Final.txt -output final-output

$ hadoop fs -get final-output rm final-output/_SUCCESS

$ hadoop jar hadoop-streaming.jar -file mapper2.py -mapper mapper2.py -input final-output/* -output final-layer-output

$ hadoop fs -cat final-layer-output/part-* > final_output.txt
```
In  the  first  map  reduce  we fetch  similar  words  in  the  name.  In  the  next  mapper  we  just  keep names which are highly common. We write the output of the map reduce to final_output.txt file As we have too many names for visualization, it is not feasible and sensible to use all the names for  visualization.  Hence,  we  use  random  100  lines  from  final  output  and  then  filter  the  files accordingly.
```
$ shuf -n 100 final_output.txt > names_list.txt module unload tensorflow/1.10-anaconda3

$ cat names_list.txt | python fetchNames.py > peoples_names_list.txt cat names_list.txt | python fetchSynset.py > synstring.txt
```
At the end of it we run the visualization code and the output is saved in visualization.png
```
$ module load tensorflow/1.10-anaconda3 
$ python Domino.py
```
#### Output Image

![image](https://github.com/shubhm-gupta/Wikipedia-Human-Names-Word-Domino/blob/master/visualization.png)

#### Domino Density  

Divided the domino into parts of 100px to 100px and measured the words in that diameter. By taking random samples of 10 such diameters we got the density as:  
The number of letters per square of the domino diameter is 8.44 letters per 100pixels diameter. The density of letters goes as high as 14 -18 words per diameter and as low as 4 to 5 words. 

## References:

[1]         Peter         Norvig,         "xkcd         Name         Dominoes",         21         March         2018, https://nbviewer.jupyter.org/github/norvig/pytudes/blob/master/ipynb/xkcd-Name- Dominoes.ipynb

