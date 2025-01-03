
# Data Pipelining and Real-Time Data Streaming with Apache Flink and Flink SQL

## Objective
This project demonstrates a comprehensive pipeline to handle streaming data by integrating Confluent schemas, Kafka topics, Apache Flink, and Elasticsearch. The primary goal is to:  
- Generate and process data using **Confluent schemas**.  
- Feed the processed data into multiple **Kafka topics**.  
- Stream and analyze the data in real-time using **Apache Flink** and **Flink SQL**.  
- Create a dashboard for visualization and derive insights using **Kibana**.
  
---

## Dataset Description

Three datasets were generated from **Confluent Avro schemas** and subsequently transformed into JSON format. The datasets represent different entities of a gaming platform:

1. **Game Rooms (bda1)**  
   - Represents information about game rooms created within the platform.  
   - Schema:  
     - `id`: Unique identifier for the game room.  
     - `room_name`: Name/type of the game room (e.g., "Classic -- Skilled").  
     - `created_date`: Timestamp of when the game room was created.

   Example Data:  
   ```json
   {"id":2975,"room_name":"Classic -- Skilled","created_date":1609459200000}
   {"id":3648,"room_name":"Arcade -- Expert","created_date":1609459400000}
   ```

2. **Player Activity (bda2)**  
   - Represents gameplay activity for each player.  
   - Schema:  
     - `player_id`: Unique identifier for the player.  
     - `game_room_id`: Game room ID where the player is playing.  
     - `points`: Points scored by the player.  
     - `coordinates`: Player's position in the game (in `[x, y]` format).

   Example Data:  
   ```json
   {"player_id":1001,"game_room_id":1529,"points":189,"coordinates":"[52,27]"}
   {"player_id":1094,"game_room_id":2675,"points":403,"coordinates":"[80,52]"}
   ```

3. **Players (bda3)**  
   - Contains information about the players registered on the platform.  
   - Schema:  
     - `player_id`: Unique identifier for the player.  
     - `player_name`: Name of the player.  
     - `ip`: IP address of the player.

   Example Data:  
   ```json
   {"player_id":1029,"player_name":"Rossie Hobben","ip":"151.228.5.139"}
   {"player_id":1076,"player_name":"Lea Murrish","ip":"34.10.81.209"}
   ```

---

## Data Pipeline Process

### 1. **Data Generation**  
   Data was generated from Confluent Avro schemas using the following commands:  
   ```bash
   ./gendata.sh gaming_games.avro xyz1.json 10000
   ./gendata.sh gaming_player_activity.avro xyz2.json 10000
   ./gendata.sh gaming_players.avro xyz3.json 10000
   ```
   - `gendata.sh`: Extracts data from the Avro schema and converts it to JSON.  
   - Input: Avro schema.  
   - Output: JSON file with random synthetic data.

### 2. **Data Transformation**  
   The JSON files were transformed into key-value pairs using the `convert.py` script to prepare them for Kafka ingestion:  
   ```bash
   python $HOME/Documents/fake/convert.py
   ```

### 3. **Kafka Ingestion**  
   Data from JSON files was streamed into Kafka topics using `gen_sample.sh`:  
   ```bash
   ./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz1.json | kafkacat -b localhost:9092 -t bda1 -K: -P
   ./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz2.json | kafkacat -b localhost:9092 -t bda2 -K: -P
   ./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz3.json | kafkacat -b localhost:9092 -t bda3 -K: -P
   ```
   - `gen_sample.sh`: Streams transformed JSON data to Kafka topics.  
   - Kafka topics created: `bda1`, `bda2`, `bda3`.

### 4. Real-Time Analysis with Apache Flink
Apache Flink and Flink SQL were utilized to perform real-time streaming analysis:
1. **Table Creation**: Tables were created in Flink SQL for each Kafka topic, specifying the schema and data format.

   - **Game Rooms Table**  
     ```sql
     CREATE TABLE game_rooms (
         id BIGINT,
         room_name STRING,
         created_date TIMESTAMP(3),
         WATERMARK FOR created_date AS created_date - INTERVAL '5' SECOND
     ) WITH (
         'connector' = 'kafka',
         'topic' = 'bda1',
         'format' = 'json'
     );
     ```

   - **Player Activity Table**  
     ```sql
     CREATE TABLE player_activity (
         player_id BIGINT,
         game_room_id BIGINT,
         points INT,
         coordinates STRING
     ) WITH (
         'connector' = 'kafka',
         'topic' = 'bda2',
         'format' = 'json'
     );
     ```

   - **Players Table**  
     ```sql
     CREATE TABLE players (
         player_id BIGINT,
         player_name STRING,
         ip STRING
     ) WITH (
         'connector' = 'kafka',
         'topic' = 'bda3',
         'format' = 'json'
     );
     ```

   - **Enriched Views**  
     Player activity was joined with game rooms and players to create a consolidated view:  
     ```sql
     CREATE VIEW enriched_player_game_activity AS
     SELECT 
         PA.player_id,
         P.player_name,
         P.ip,
         PA.game_room_id,
         GR.room_name,
         GR.created_date,
         PA.points,
         PA.coordinates
     FROM player_activity AS PA
     LEFT JOIN game_rooms AS GR
         ON PA.game_room_id = GR.id
     LEFT JOIN players AS P
         ON PA.player_id = P.player_id;
     ```

### 5. **Data Ingestion into Elasticsearch**  
   The enriched data was stored in an Elasticsearch index for visualization:  
   ```sql
   CREATE TABLE player_game_dashboard (
       player_id BIGINT,
       player_name STRING,
       ip STRING,
       game_room_id BIGINT,
       room_name STRING,
       created_date BIGINT,
       points INT,
       coordinates STRING
   ) WITH (
       'connector' = 'elasticsearch-7',
       'hosts' = 'http://elasticsearch:9200',
       'index' = 'player_game_dashboard',
       'format' = 'json'
   );
   ```

### 5. **Dashboard Creation with Kibana**
   The data was visualized using Kibana by creating an index (`player_game_dashboard`) and designing a detailed dashboard.
   
---

# Dashboard Analysis

![Docker_Flink](https://github.com/user-attachments/assets/12c2099b-31aa-4b02-b07f-e317c3fc5346)

---

### Video

https://github.com/user-attachments/assets/238a5acd-f25d-4616-a41e-c73116bcd10e

---

#### **1. Unique Players Count (Gauge)**
- **Observation:** The platform has a total of 3,293 unique players.
- **Insights:** 
  - This indicates a healthy player base and engagement.
  - With such a large number of players, the platform has significant opportunities to build a community, run targeted campaigns, and introduce loyalty programs.

---

#### **2. Maximum and Minimum Points (Gauges)**
- **Maximum Points:** The highest score achieved by any player is 499.  
- **Minimum Points:** The lowest score recorded in the game is 10.
- **Insights:** 
  - The wide range of scores reflects the diversity in player skill levels, from beginners to highly skilled players.
  - Newer or less skilled players may require additional tutorials or practice modes to enhance their performance and engagement.

---

#### **3. Top 5 Most Popular Game Rooms (Gauge)**
- **Observation:** Game rooms with IDs 898, 894, 868, 859, and 847 are the most popular.
- **Insights:** 
  - These rooms attract the most players and should be prioritized for updates, better features, or exclusive events.
  - Analyzing the features of these rooms can provide insights into what players enjoy most, which can inform the design of new rooms.

---

#### **4. Top Ranked Players (Bar Chart)**
- **Observation:** Players such as Riva Rossant, Lucas Midson, and Zeb Ryllat rank among the top based on maximum points scored.
- **Insights:**
  - These players can be recognized and rewarded to maintain their interest in the game.
  - Highlighting their achievements in a leaderboard or hall of fame can foster competition and motivation among other players.

---

#### **5. Average Points Across Different Room Types (Bar Chart)**
- **Observation:** Average points scored vary slightly across room types like "Arcade -- Skilled," "Survival -- Rookie," and others.
- **Insights:**
  - Room types with higher average scores may be perceived as more rewarding and engaging, encouraging player retention.
  - Rooms with lower scores may need adjustments, such as reduced difficulty levels or enhanced incentives.

---

#### **6. Distribution of Room Types (Pie Chart)**
- **Observation:** The distribution shows the percentage of activity across different room types, with each type contributing significantly to the overall platform.
- **Insights:**
  - A balanced distribution indicates diverse player preferences.
  - Understanding which room types are less utilized can guide promotional efforts to increase their popularity.

---

#### **7. Unique Player Trend in Arcade Rooms (Line Chart)**
- **Observation:** The number of unique players fluctuates daily across different Arcade room types (e.g., "Arcade -- Skilled," "Arcade -- Rookie," "Arcade -- Expert").
- **Insights:**
  - Peaks in the trend may coincide with special events or promotions, suggesting the importance of planned engagement activities.
  - Declines in player trends could be addressed with reminders, notifications, or time-bound rewards to re-engage players.

---

#### **8. Average of Top 3 Players Across Different Room Types (Bar Chart)**
- **Observation:** Players like Cyril Yellowlas, Sheridan Foulsham, and Gustave Westhofer have consistently high average points across room types.
- **Insights:**
  - High-performing players can act as ambassadors for the game, promoting it to their networks.
  - Offering tier-based rewards or competitions among top players can further increase their engagement.

---

#### **9. Top 3 Players with the Highest Room Creation (Bar Chart)**
- **Observation:** Players like Hayley Tuma, Adele Bohl, and Constantin Mears have created the most game rooms.
- **Insights:**
  - Encouraging these active creators with room customization options or creator badges can boost their involvement and incentivize others to create rooms.

---

#### **10. Top 3 Players’ Room Creation Across Different Room Types (Stacked Bar Chart)**
- **Observation:** The most rooms are created in categories like "Survival -- Expert" and "Arcade -- Skilled."
- **Insights:**
  - These categories could be used as benchmarks to design similar rooms that encourage participation and engagement.

---

#### **11. Room Creation Trend of Top 3 Room Types (Line Chart)**
- **Observation:** Daily trends show variations in room creation across popular room types like "Arcade -- Expert," "Classic -- Skilled," and "Classic -- Expert."
- **Insights:**
  - Declines in creation trends may indicate that players are exploring other game modes.
  - Room creation campaigns, such as discounts or bonus rewards for room creation, could encourage users to create rooms consistently.

---

### **Managerial Insights**

1. **Player Engagement:**
   - The platform has a strong base of 3,293 players. Retention strategies like loyalty rewards, tournaments, or exclusive game modes should be implemented to keep players engaged.
   - Encourage top players by offering incentives such as leaderboards, badges, or exclusive access to premium rooms.

2. **Room Optimization:**
   - Focus resources on maintaining and enhancing the most popular game rooms, especially those identified as top performers.
   - For rooms with lower scores or engagement, analyze player feedback to identify potential improvements in design or mechanics.

3. **Trends and Patterns:**
   - Use the daily trends to schedule marketing campaigns during periods of declining engagement.
   - Promote room creation in underrepresented categories through limited-time offers or gamified rewards.

4. **Data-Driven Expansion:**
   - Leverage insights about popular rooms and player preferences to design new rooms or game features.
   - Consider cross-promotions or collaborations targeting specific demographics based on activity patterns.

---

### **Learnings**

1. **Understanding Player Behavior:**
   - The dashboard highlights key aspects of player behavior, such as top scorers, room creators, and overall engagement trends. This can guide future decisions to enhance player satisfaction.

2. **Real-Time Data Utilization:**
   - The integration of Kafka, Elasticsearch, and Kibana enables real-time data streaming and visualization, providing actionable insights promptly.

3. **Performance Monitoring:**
   - Regularly tracking metrics like unique player counts, trends in room creation, and score distributions helps in identifying areas for improvement or expansion.

4. **Strategic Planning:**
   - The dashboard aids in planning promotions, designing new features, and allocating resources effectively by providing data-driven insights.

5. **Effective Visualizations:**
   - The use of varied visualizations (gauges, bar charts, pie charts, and line charts) ensures a comprehensive understanding of data from different perspectives.

---
