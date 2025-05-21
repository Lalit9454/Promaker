# Promaker
!pip install openai==1.12.0

!pip install gradio==4.19.0

from bs4 import BeautifulSoup
import pandas as pd
from openai import OpenAI
import gradio as gr
from google.colab import userdata

client = OpenAI(api_key=userdata.get('OPENAI_API_KEY'))

def pull_match_data(m_link):
    r = requests.get(m_link)
    soup = BeautifulSoup(r.text, 'html.parser')

    batting_columns = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto ci-scorecard-table')[:1]:
        for j in i.find_all('th'):
            batting_columns.append(j.text)
    
    batting = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto ci-scorecard-table')[:1]:
        rows = i.find_all('td', class_='ds-min-w-max')
        for i in range(0, len(rows), 8):
            batter = []
            for j in range(i, i+8):
                try:
                    batter.append(rows[j].text)
                except IndexError:
                    break
            batting.append(batter)
    innings1_batting = pd.DataFrame(batting, columns=batting_columns)

    batting_columns = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto ci-scorecard-table')[1:2]:
        for j in i.find_all('th'):
            batting_columns.append(j.text)
    
    batting = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto ci-scorecard-table')[1:2]:
        rows = i.find_all('td', class_='ds-min-w-max')
        for i in range(0, len(rows), 8):
            batter = []
            for j in range(i, i+8):
                try:
                    batter.append(rows[j].text)
                except IndexError:
                    break
            batting.append(batter)
    innings2_batting = pd.DataFrame(batting, columns=batting_columns)

    bowling_columns = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto')[:1]:
        for j in i.find_all('th'):
            bowling_columns.append(j.text)
    
    bowling = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto')[:1]:
        rows = i.find_all('td')
        bowler = []
        for i in range(0, len(rows)):
            if len(bowler) < 11:
                if len(rows[i].text) < 20:
                    try:
                        bowler.append(rows[i].text)
                    except IndexError:
                        break
            else:
                bowling.append(bowler)
                bowler = []
                if len(rows[i].text) < 20:
                    bowler.append(rows[i].text)
    innings1_bowling = pd.DataFrame(bowling, columns=bowling_columns)

    bowling_columns = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto')[1:2]:
        for j in i.find_all('th'):
            bowling_columns.append(j.text)
    
    bowling = []
    for i in soup.find_all(class_='ds-w-full ds-table ds-table-md ds-table-auto')[1:2]:
        rows = i.find_all('td')
        bowler = []
        for i in range(0, len(rows)):
            if len(bowler) < 11:
                if len(rows[i].text) < 20:
                    try:
                        bowler.append(rows[i].text)
                    except IndexError:
                        break
            else:
                bowling.append(bowler)
                bowler = []
                if len(rows[i].text) < 20:
                    bowler.append(rows[i].text)
    innings2_bowling = pd.DataFrame(bowling, columns=bowling_columns)

    return innings1_batting, innings1_bowling, innings2_batting, innings2_bowling

def get_completion(m1_link, m1_batting_team, m1_bowling_team, m2_link, m2_batting_team, m2_bowling_team, m3_team1, m3_team2, model="gpt-4-turbo-2024-04-09"):
    m1_innings1_batting, m1_innings1_bowling, m1_innings2_batting, m1_innings2_bowling = pull_match_data(m1_link)
    m2_innings1_batting, m2_innings1_bowling, m2_innings2_batting, m2_innings2_bowling = pull_match_data(m2_link)

    system_prompt = """
    You are a huge cricket fan, and good with analysing match data.
    """

    prompt = """
    I will provide you batting & bowling data for two cricket matches.
    Match 1 data provided between three backticks.
    Match 2 data provided between separator ####.
    Now, one team from each of these two matches are playing tonight.
    Your task is to analyse the two matches data.
    And create a Fantasy team for tonight's match between {m3_team1} & {m3_team2}.
    This fantasy team should have 11 players.

    ```

    Match 1: {m1_batting_team} vs {m1_bowling_team}
    Innings 1 >> {m1_batting_team} Batting
    {m1_innings1_batting}
    Innings 1 >> {m1_bowling_team} Bowling
    {m1_innings1_bowling}
    Innings 2 >> {m1_bowling_team} Batting
    {m1_innings2_batting}
    Innings 2 >> {m1_batting_team} Bowling
    {m1_innings2_bowling}
    ```

    ####

    Match 1: {m2_batting_team} vs {m2_bowling_team}
    Innings 1 >> {m2_batting_team} Batting
    {m2_innings1_batting}
    Innings 1 >> {m2_bowling_team} Bowling
    {m1_innings1_bowling}
    Innings 2 >> {m2_bowling_team} Batting
    {m2_innings2_batting}
    Innings 2 >> {m2_batting_team} Bowling
    {m2_innings2_bowling}
    ####
    """

    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": prompt}
    ]

    response = client.chat.completions.create(
        model="gpt-4-turbo-2024-04-09",
        messages=messages,
        temperature=0
    )

    return response.choices[0].message.content

iface = gr.Interface(
    fn=get_completion,
    inputs=[
        gr.Textbox(label="Team 1 Last Match Scorecard", lines=1
