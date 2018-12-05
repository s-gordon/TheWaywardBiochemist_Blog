---
layout: post
title: "PhD Thesis Run-Down"
date: 2018-11-22 14:34:00 +1100
comments: true
sharing: true
categories: Science
featured-img: path
tags:
- python
- xkcd
- statistics
- latex
- paper
- programming
- unix
---

After wrapping up my PhD thesis, it seemed like a fun idea to analyse my writing tendencies over the time course of the thesis. Some questions were: How did writing productivity change over time? Were there dips and peaks? Was there a magical golden hour where the words just came pouring out? The idea came from a related [blog post](https://matthieu.io/blog/2018/02/28/phd-post-mortem/).

I wrote my thesis in [LaTeX](https://www.latex-project.org/) using [Git](https://git-scm.com/) for version control and [Matplotlib](https://matplotlib.org/) for constructing graphs. This means that I can roll through my git commit history and collect various statistics to study my writing habits. [TeXcount](http://app.uio.no/ifi/texcount/) does most of the hard work in extracting details from TeX files. Put in a shell script, it looks something like this.

```bash
# call using
#   $ bash script_name.sh origin/<branch>

# outfile
o="./word_usage_commits.txt"

# define headers for outfile and to stdout
printf "%s,%s,%s,%s,%s,%s,%s,%s\n" "commit" "date" "time" "total words" "words outside text" "headers" "floats" "equations" > "$o"
printf "%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\n" "commit" "date" "time" "total words" "words outside text" "headers" "floats" "equations"

# roll through the git history
# for each commit, extract wordcounts and other metrics
while read -r rev; do
    git checkout --quiet "$rev"
    if ! texcount -inc -merge -total -dir=tex tex/main.tex >/dev/null; then
      >&2 echo "Commit $rev failed"
      exit 1
    else
      tc=$(texcount -inc -merge -total -dir=tex tex/main.tex)
      # total words
      tot=$(echo "$tc" | grep "Words in text" | awk '{print $4}')
      # Words outside text
      words_outside=$(echo "$tc" | grep "Words outside text" | awk '{print $6}')
      # number of headers
      n_headers=$(echo "$tc" | grep "Number of headers" | awk '{print $4}')
      # number of floats
      floats=$(echo "$tc" | grep "Number of floats" | awk '{print $4}')
      # number of equations
      equations=$(echo "$tc" | grep "Number of math inlines" | awk '{print $5}')
      # date/time
      date=$(git show -s --format=%ci | awk '{print $1}')
      time=$(git show -s --format=%ci | awk '{print $2}')
      printf "%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\n" "$rev" "$date" "$time" "$tot" "$words_outside" "$n_headers" "$floats" "$equations"
      printf "%s,%s,%s,%s,%s,%s,%s,%s\n" "$rev" "$date" "$time" "$tot" "$words_outside" "$n_headers" "$floats" "$equations" >> "$o"
    fi
done < <(git rev-list "$1")
```

This generates a plain text file with all the metrics needed to piece together my writing habits. Lovely.

Using python, I can easily plot total words as a function of commit date/time.

```python
import matplotlib as mpl
import matplotlib.pyplot as plt
import pandas as pd

# input file containing commit history in the form:
# commit,date,total words,words outside text,headers,floats,equations
# 03161ca9facbce55ab90b4fe3dcb152735eb68be,2018-09-12-13:14:10,29959,6951,121,111,279
ifile = 'commits.txt'

# xkcd filter. for giggles
plt.xkcd()

fig, ax = plt.subplots()
ax.spines['right'].set_color('none')  # remove right aspect of plot frame
ax.spines['top'].set_color('none')  # remove top aspect of plot frame
ax.tick_params(axis=u'both', which=u'both', length=0)  # hide irrelevant ticks
ax.set_ylabel('words')
plt.xticks(rotation=90)  # rotate x-axis tick labels

df = pd.read_csv(ifile, header=0)  # read in file to dataframe
df = df.iloc[::-1]  # commit history is in reverse order. invert
df.index = range(len(df))  # reset index
df['change'] = df['total words'].diff()  # calculate change in word count between commits

ax.plot_date(df['date'], df['total words'], '-')
```

![Words by date](/assets/img/posts/thesis_word_count_by_date.png)

Neat, huh? Can really see that I hunkered down from late-2017 to build up that word count. We can also group word additions/subtractions by the day of the week.

```python
import matplotlib as mpl
import matplotlib.pyplot as plt
import dateparser as dparse

def sum_weekday(df, day, daycolname="dayofweek", sumcolname="change"):
    """Docstring for sum_weekday.
    df: pandas dataframe
    day: day of week to search for
    colname: name of pandas dataframe column containing day of week info
    wordcolname: name of pandas dataframe column to sum

    """
    return df.loc[df[daycolname] == day][sumcolname].sum()

# xkcd filter. for giggles
plt.xkcd()

fig, ax = plt.subplots()
ax.spines['right'].set_color('none')  # remove right aspect of plot frame
ax.spines['top'].set_color('none')  # remove top aspect of plot frame
ax.tick_params(axis=u'both', which=u'both', length=0)  # hide irrelevant ticks
ax.set_ylabel('words')
plt.xticks(rotation=90)  # rotate x-axis tick labels

df = pd.read_csv(ifile, header=0)  # read in file to dataframe
df = df.iloc[::-1]  # commit history is in reverse order. invert
df.index = range(len(df))  # reset index

# define date parser for the input file date/time format
date = [dparse.parse(
    d, date_formats=['%Y-%m-%d-%H:%M:%S']) for d in df['date']]
df['date'] = date

# from the date/time, establish day of week for each commit
df["dayofweek"] = df["date"]
df["dayofweek"] = df['dayofweek'].apply(lambda x: x.strftime('%A'))
days = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday',
        'Saturday', 'Sunday']
plt.xticks(range(7), dayweek["Day"])

# group like-day commits and sum
dayweek = pd.DataFrame({"Day": days,
                        "Words": [sum_weekday(df, d) for d in days]})

# finally, plot
ax.bar(range(7), dayweek["Words"], align='center', color="white")
```

![Words by week day](/assets/img/posts/thesis_word_count_by_day.png)

Can clearly see that Thursday was the workhorse of the group, while I took it a bit easy over weekends.

How about looking at the data as a function of time of day?

```python
import matplotlib as mpl
import matplotlib.pyplot as plt
import dateparser as dparse

# xkcd filter. for giggles
plt.xkcd()

fig, ax = plt.subplots()
ax.spines['right'].set_color('none')  # remove right aspect of plot frame
ax.spines['top'].set_color('none')  # remove top aspect of plot frame
ax.tick_params(axis=u'both', which=u'both', length=0)  # hide irrelevant ticks
ax.set_ylabel('words')
ax.set_xlabel("hour")
plt.xticks(rotation=90)  # rotate x-axis tick labels

df = pd.read_csv(ifile, header=0)  # read in file to dataframe
df = df.iloc[::-1]  # commit history is in reverse order. invert
df.index = range(len(df))  # reset index

# define date parser for the input file date/time format
date = [dparse.parse(
    d, date_formats=['%Y-%m-%d-%H:%M:%S']) for d in df['date']]
df['date'] = date

# change first entry delta words from NaN to 0
for index in range(0, 1):
    df.ix[index, 'change'] = 0

df = df.reset_index().set_index('date')  # re-assign index to date column
df = df["change"].resample("H").sum()  # define hour blocks for each commit
t = pd.DatetimeIndex(df.index)
p = df.groupby([t.hour]).sum()  # sum commits with same time blocks

# finally, plot
p.plot(kind='bar', color="white")
```

![Words by time of day](/assets/img/posts/thesis_word_count_by_hour.png)

Finally, how do the number of figures/tables trend with time? Just have to substitute `df['total words']` for `df['floats']` in the original python code block.

![Words by date](/assets/img/posts/thesis_float_count_by_date.png)

Not surprisingly, these seem to correlate well with word count.

If you have any opinions or ideas, let me know in the comments section below!
