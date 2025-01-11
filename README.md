           Football players ranked by their international goal records

This dataset provides a detailed overview of football players who have achieved remarkable goal-scoring records in international matches. It includes the following key attributes:
1.	Rank: Indicates the player’s position in terms of total international goals scored.
2.	Player: Name of the footballer.
3.	Nation: The country they represent in international football.
4.	Confederation: The football governing body under which their nation falls (e.g., UEFA, AFC).
5.	Goals: Total international goals scored by the player.
6.	Caps: The number of matches (appearances) the player has made for their national team.
7.	Goals per match: The average number of goals scored per game.
8.	Career span: The period during which the player was active in international football.
9.	Date of 50th goal: The specific date when the player scored their 50th international goal.
The dataset consists of 82 players, offering insights into their performance across different countries and confederations

CODE
__________________________________________________________________________________________________________________________________


import pandas as pd
import numpy as np
import json
import threading
import requests
import matplotlib.pyplot as plt
import os

file_path = 'dataset.csv'
if not os.path.exists(file_path):
    print(f"Dataset file not found at {file_path}")
    exit()

try:
    dataset = pd.read_csv(file_path, encoding='latin1')
    print("Dataset loaded successfully!")
except Exception as e:
    print(f"An error occurred while loading the dataset: {e}")
    exit()

print("Columns in the dataset:", dataset.columns)


def preprocess_data(df):
    def split_career_span(span):
        try:
            if isinstance(span, str) and '-' in span:
                start, end = span.split('-')
                start = int(start) if start else None
                end = int(end) if end else None
                return start, end if end else (start, None)
            return None, None
        except ValueError as e:
            print(f"Error splitting career span '{span}': {e}")
            return None, None

    df['Career start'], df['Career end'] = zip(*df['Career span'].apply(split_career_span))

    df['Date of 50th goal'] = pd.to_datetime(df['Date of 50th goal'], format='%Y-%m-%d', errors='coerce')

    if 'Nation' in df.columns:
        df['Nation'] = df['Nation'].str.strip()
    return df


try:
    dataset = preprocess_data(dataset)
except Exception as e:
    print(f"Error during preprocessing: {e}")
    exit()


def save_to_json(df, file_name):
    try:
        df.to_json(file_name, orient='records', lines=True)
        print(f"Data successfully saved to {file_name}")
    except Exception as e:
        print(f"Error saving data to JSON: {e}")

save_to_json(dataset, 'processed_data.json')

def scrape_player_stats(player_name):
    url = f"https://en.wikipedia.org/wiki/{player_name.replace(' ', '_')}"
    try:
        response = requests.get(url, timeout=10)
        if response.status_code == 200:

            content_preview = response.content[:100].decode('utf-8', errors='ignore')
            return {"Player": player_name, "Content": content_preview}
        else:
            return {"Player": player_name, "Error": f"HTTP {response.status_code}"}
    except requests.exceptions.RequestException as e:
        return {"Player": player_name, "Error": f"Request failed: {str(e)}"}

def scrape_multiple_players(players):
    results = []
    threads = []

    def worker(player, idx):
        result = scrape_player_stats(player)
        if isinstance(result, dict):
            results[idx] = result
        else:
            results[idx] = {"Player": player, "Error": "Failed to scrape"}

    for idx, player in enumerate(players):
        results.append(None)
        thread = threading.Thread(target=worker, args=(player, idx))
        threads.append(thread)
        thread.start()

    for thread in threads:
        thread.join()

    return results


if 'Player' in dataset.columns:
    players = dataset['Player'].dropna().head(5).tolist()
    if players:
        scraped_data = scrape_multiple_players(players)

        if scraped_data:
            try:
                with open('scraped_data.json', 'w') as f:
                    json.dump(scraped_data, f, indent=4)
                print("Scraped data saved successfully.")
            except TypeError as e:
                print(f"Error saving scraped data: {e}")
        else:
            print("No valid data was scraped.")
    else:
        print("No valid players found in the dataset for scraping.")
else:
    print("Column 'Player' not found in the dataset.")


def plot_top_25_players_by_goals(df):
    if 'Goals' in df.columns and 'Player' in df.columns:
        df_clean = df.dropna(subset=['Goals', 'Player'])

        df_sorted = df_clean.sort_values(by='Goals', ascending=False).head(25)

        if not df_sorted.empty:
            df_sorted['Rank'] = range(1, len(df_sorted) + 1)

            print("\nTop 25 Players by Most Goals (Rank and Goals):")
            print(df_sorted[['Rank', 'Player', 'Goals']])

            plt.figure(figsize=(14, 8))
            plt.bar(df_sorted['Player'], df_sorted['Goals'], color='skyblue')
            plt.title('Top 25 Players by Most Goals')
            plt.xlabel('Player')
            plt.ylabel('All-time Goals')
            plt.xticks(rotation=45)
            plt.tight_layout()
            plt.show()
        else:
            print("No valid data available to plot.")
    else:
        print("Required columns 'Goals' or 'Player' are missing from the dataset.")

plot_top_25_players_by_goals(dataset)

print("Project workflow complete. Processed data saved as 'processed_data.json' and scraped data saved as 'scraped_data.json'.")
