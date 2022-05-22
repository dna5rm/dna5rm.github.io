---
title: 'Quiz a user from a question pool in a txt, tab seperated, spreadsheet.'
date: '2011-01-24T17:37:45+00:00'
author: deaves
layout: post
permalink: /2011/01/quiz-a-user-from-a-question-pool-in-a-txt-tab-seperated-spreadsheet/
categories:
    - 'Linux Scripts'
tags:
    - csv
    - linux
    - quiz
---

```bash
#!/bin/bash
## Created by: deaves
# Quiz a user from a question pool in a txt, tab seperated, spreadsheet.
#
# The following is how to format the rows in the input file:
#   ColumnA [Integer]: True Answer (D=0,E=1,F=2,...)
#   ColumnB [String]:  Answer Number
#   ColumnC [String]:  Question
#   ColumnD+[String]:  Answer
#
# FYI: If you have problems try using Open Office to create the spreadsheet,
# MS Office does not properly suround cells with quotes so it gives eval problems.
#
## Requires: dialog (Optional: TTS synthesizer)

## Comment out speakCMD var if no text/speach synth.
#speakCMD="flite -s duration_stretch=0.8"
#speakCMD="xclip -i -selection clipboard"

### Begin Functions ###
function diaSTRING () {
  ### Retrun no variables; just create a dialog string.
  printf "dialog --stdout --backtitle \"${cTALLY}/${qTALLY}:\t${backTITLE}\" \\
   --title \"${qArray[1]}\" --menu \'${qArray[2]}\' 0 0 0 "

  # Count the total answer elements in the array.
  ansNUM=$(expr ${#qArray[@]} - 3)
  count=0    # The answer number the user sees.
  acount=2   # The true answer element in the array.

  # Generate the possible answers.
  while [ "${count}" -lt ${ansNUM} ] && [ -n "${qArray[${acount}]}" ]
  do
    let acount++
    printf "${count} \'${qArray[${acount}]}\' "
    let count++
  done

  unset count acount ansNUM   # Unset variables.
}


function diaRESULT () {
  ### Display the test results...

  # Display number of questions answered correctly when finished.
  if [ -z "${dANS}" -a -z "${REVIEW}" ]; then
    dialog --backtitle "${backTITLE}" --title "${quizFILE}" \
     --msgbox "You answered:\n ${cTALLY} questions out of ${qTALLY} correct" 0 0

  # Display correct answer if the wrong answer was selected.
  else
    dialog --backtitle "${backTITLE}" --title "${qArray[1]}" \
     --msgbox "Q: $(echo "${qArray[2]}")\n\nA: $(echo "${qArray[`expr ${qArray[0]} + 3`]}")" 0 0
  fi
}


function pick_Question () {
  ### Select question and build qArray...
  if [ -z ${QUESTION} ]; then
    QUESTION=$[ ( ${RANDOM} % ${QUESTIONS} ) + 1 ]

    # Do not ask the same question twice...
    [ -z ${lastQUESTION} ] && { lastQUESTION=0; }
    while [ ${QUESTION} = ${lastQUESTION} ]; do
      QUESTION=$[ ( ${RANDOM} % ${QUESTIONS} ) + 1 ]
    done

    lastQUESTION=${QUESTION}  # Remember the previous question.
  fi

  # Use eval to load the entire row into qArray...
  eval qArray=( `sed -n ${QUESTION}'p' "${quizFILE}" | 's/%/%%/g'` )

  # Unset the current randomly chosen question.
  [ -z ${REVIEW} ] && { unset QUESTION; }
}


function review_Questions () {
  ### Review each question in the entire pool.
  declare -i QUESTION=1

  while [ "${QUESTION}" != `expr ${QUESTIONS} + 1` ]; do
    pick_Question        # Load question into qArray.
    dANS="${qArray[0]}"  # Set the correct answer.
    speakTEXT &          # Read Q&A to the user.
    diaRESULT            # Display Q&A to the user.

    unset dANS qArray    # Unset variables.
    let QUESTION++       # Next question.
  done
}


function speakTEXT () {
  ### Use a text to speach synth to read the question & answer.
  ## If this function is unwanted leave the speakCMD variable blank.

  # Ask the question...
  if [ -n "${speakCMD}" ] && [ -z "${dANS}" ]; then
    echo "${qArray[2]}" | ${speakCMD}

  # Review the question & answer...
  elif [ -n "${speakCMD}" ] && [ -n "${REVIEW}" ]; then
    echo "Question, ${qArray[1]} . ${qArray[2]}" | ${speakCMD}
    echo "The Answer is, $(echo ${qArray[`expr ${qArray[0]} + 3`]})" | ${speakCMD}

  # Give the answer...
  elif [ -n "${speakCMD}" ]; then
    echo "The correct answer was: $(echo ${qArray[`expr ${qArray[0]} + 3`]})" | ${speakCMD}
  fi
}


function usage () {
  ### Display the script arguments.
  printf "Usage: $0 -f <file .txt=""> [-rs]\n\n"
  printf "Required...\n"
  printf "\t-f: Specify the file containing the question pool. (required)\n\n"
  printf "Optional...\n"
  printf "\t-r: Review all the questions in the pool.\n"
  printf "\t-s: Disable sound (if the speakCMD is defined).\n\n"
}


### Get passed arguments. ###
while getopts ":f:rs" ARG; do
  case "${ARG}" in
    f) quizFILE="${OPTARG}";;
    r) REVIEW="Y";;
    s) unset speakCMD;;
  esac
done

### Display command usage and exit ###
[ -z "${quizFILE}" ] && { usage && exit 1; }

### Variables that should not be changed ###
backTITLE="Dave's cute quiz script."
QUESTIONS=`wc -l "${quizFILE}" | awk '{print $1}'`
qTALLY=0   # Total questions asked...
cTALLY=0   # Total questions correct...


### Review all questions and exit script ###
[ "${REVIEW}" = "Y" ] && { review_Questions; exit 0; }

### Main Loop ###
while true; do
  pick_Question                               # Pick a Question.
  speakTEXT &                                 # Read the question.
  dANS=`bash -c "$(diaSTRING)"`               # Ask question and store response as "dANS".

  [ -z "${dANS}" ] && { diaRESULT && break; } # Break out of the loop if user selects cancel.
  let qTALLY++                                # Tally the total number of questions asked.

  if [ ${dANS} == "${qArray[0]}" ]; then      ### Response is correct ###
    let cTALLY++                              # Tally the questions answered correct.

  else                                        ### Response is wrong ###
    speakTEXT &                               # Speak the correct answer.
    diaRESULT                                 # Show the correct answer.
  fi

  unset dANS qArray                           # Unset loop variables at end of pass.
done

### Finish ###
```
