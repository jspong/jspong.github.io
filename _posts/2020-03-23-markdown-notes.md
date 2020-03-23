---
title: "Using Git and Markdown to Take Daily Notes for Self-Esteem and Profit."
layout: post
tags: [productivity,timetracking,git]
---
# Using Git and Markdown to Take Daily Notes for Self-Esteem and Profit.

I've been struggling for a long time with staying focused: time tracking, note-taking, goal setting, finalizing tasks were all frequent stumbling blocks between myself and actually delivering results.

One of these activities in particular has seemed to encourage and nurture the others: note taking. I can't not pay attention to what I'm doing if I'm writing about it. I can't pretend I don't waste time -- or that I haven't been spinning my wheels on the same problem -- if I have a written record of me doing it for weeks, or months. Or years. 

So I started taking notes of everything I do, why I do it, what I struggle with, what I want to do. I started to write whatever I even question whether I should write it. This showed that nearly every thought has some sort of value in being written down.

I then started to want to keep things around, so I decided to keep a `git` repo of my writings. To make it quick, I made a function in my `.zsrhc` file so I could quickly jot down my thoughts just by typing `notes` into a terminal.

```bash
function notes() {
  pushd ${HOME}/github/jspong/notes                                    # Do the work in my github repo
  git commit --all --message="Cleaning up lingering notes"             # Commit any changes I made not using this function

  TODAY=$(date +'%F')                                                  # Get today's date in YYYY-MM-DD format
  NOTES=${TODAY}.md                                                    # Use the date for the filename

  vim ${NOTES}                                                         # Edit the file

  git commit --all --message="Notes for ${TODAY}" 2>&1 >/dev/null      # Check in my changes to git
  popd                                                                 # Return to the directory I called the function from
}
```

This was great, but the more notes I took the more notes I wanted to read. Yesterday's notes meant knowing where the notes were, and opening that up by directory. Looking up the date for three days ago was sufficient hassle that I wanted to automate it away.

I knew there had to be a conveniently clever fuzzy date library for python, and I found one in [dateparser](https://dateparser.readthedocs.io/en/latest/). With the following changes, I could say `notes yesterday`, or `notes '4 days ago'`:

```bash
if [[ -n "$1" ]]                                                                       # Did we pass an argument to notes?
then
  DATE=$(python -c "import dateparser; print(dateparser.parse('$1').strftime('%F'))")  # Convert the fuzzy date to YYYY-MM-DD
  if [ $? -eq 1 ]                                                                      # If parse doesn't understand the date,
  then                                                                                 #   it returns None, causing an AttributeError
    popd                                                                               #   causing the python interpreter to exit 1
    return 1
  fi
  MSG="Notes for '$1' ($DATE)")
else
  DATE=$(date +'%F')                                                                  # If there was no argument, just edit today's notes
  MSG="Notes for $DATE"
fi
NOTES=${DATE}.md
# ...
```

This has been going great, and the benefits of writing things down as I go have blown me away. I still forget things, and I wish I had a good task tracking system still. Being a well-intentioned, if perhaps a bit self-delusional, software engineer, I frequently leave "todo: " comments in my notes. So now I want a todo list document that can update itself without too much work from me.

```bash
# ...
touch ${NOTES}     # Create file if it doesn't exist
git add ${NOTES}   # Add it to git if it's not there
vim ${NOTES}
git diff | grep +todo: | sed "s/+todo:/- [ ] (${DATE}):/" >> todo.md
# ...
```

Using [grip](https://github.com/joeyespo/grip), I can keep `todo.md` automatically rendering in my browser, with a live to-do list that automatically updates from my periodic thoughts. Now what I need is a way to enable the "done" checkboxes in-browser that automatically update my Markdown.

todo: I think a better solution will be to put some sort of `done (taskid)` syntax into my notes so that I can automatically tick the boxes when I write notes too.
