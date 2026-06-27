# home-assistant-ml

My first venture into machine learning.
The goal is to allow home assistant to actually make proper predictions and learn from users.


> [!NOTE] Note on LLM usage in this project
>  I am personally not very familiar with machine learning, so I will probably use some LLM generated code here and there. This is first and foremost a learning project for me though, so the actually relevant parts will likely be done by hand.
> So far, everything here is done by hand.

## Concept (High Level)

Any entity with State in HASS[^hass] can be made into a learning entity, but note that each entity essentially gets its own lightweight[^lightweight] ml model.

I like having an example, so let's say we're in a 1 room apartment that has a `light.main_light` and a bunch of sensors. Now, we want the light to learn from our actions.

For that, we first define an entity config:
[Example Config](/docs/examples/entity_config_example.jsonc).
This tells the supervisor 3 things:
- Which sensors[^sensors] should the model use as input parameters for its prediction?
- Which of those sensors should _trigger_ a prediction?
- Which state should actually be predicted?

From that data, the script will create many dataframes, extracted from an InfluxDB source. The more history, the better.

### Dataframes

Each dataframe contains the state of all sensors when the triggering change occurs, as well as the (prediction relevant) state after a set delay (e.g. 30 seconds).

**Why only record state after 30s?** \
Say you turn on a light with a switch. But its late at night, so it's way too bright at 100%. You then dim it down to 30%, and leave it there.
Without the delay, we'd get 70 dataframes for each dim step between 100% and 30%. That'd confuse the model and lead to random predictions.
By only recording the light's state after 30 seconds, we can be pretty sure that the state recorded is the state the user actually wants the light to be in, instead of some in-between state.
> An idea for improvement I had here was to wait for the state to be idle for a few seconds instead of a set delay, but I figured that might be too much work for too little improvement. I'll see once I have the actual data.

### Implementation stages

#### 1. Training

Pretraining will extract as many dataframes as possible from the InfluxDB, and then start a supervised training session on those to create a Random Forest model.

#### 2. Proxy Testing

To get the latest habits and training data, the model can first be ran in a "Proxy Environment". The program will create a proxy entity for your learning entity and apply its predictions there. Then, after 30s, it'll check the proxy's state against the actual entity state and note the deviation, and also record the new dataframe.

#### 3. Online Learning

Once the accuracy is satisfying enough, the model can be deployed. Every time it's triggered, it'll apply it's prediction to the entity, and of course record the dataframe 30 seconds later with its deviation.

[^hass]: Here: Home Assistant
[^sensors]: Note that sensors here do not explicitly need to be HASS Sensors. Any value, state or template can be a sensor.
[^lightweight]: Even lightweight is an overstatement. Models with 10 or even 50 input sensors predicting all the state data for even many entities at a time are minuscule, kilo- or megabytes in size and take practicly no time to infer. Training these models with 10000 dataframes will take minutes on any decently modern CPU.