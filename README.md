# Tesortrade
Songhao Li
Modified from https://github.com/notadamking/tensortrade 

Tensortrade is a modulized quantitative reinforcement learning project. According to my adjustment, this project has achieved self-provided data input, futures market environment configuration, and deployment of tensorforce agents incluing PPO and A2C.

For a preview of the learning results using PPO algorithm (https://arxiv.org/abs/1707.06347):
![png1](https://github.com/EconomistGrant/HTFE-tensortrade/blob/master/Data/Picture1.png)
![png2](https://github.com/EconomistGrant/HTFE-tensortrade/blob/master/Data/Picture2.png)
The readme file will consist of the following parts: 
Modules Overview  | Running the Program | Details of Modules | Case Analysis (PPO/A2C)

# Modules Overview
There are two major modules: trading environment and learning agent. Trading environment is managed by environment.TradingEnvironment, including the following sub-modules:
1. Exchanges (state space)
2. Actions (action space)
3. Reward 
4. Trade (support data flow in the program)
This four sub-moduls have been expanded with new classed to fit futures market environment; there will be specified introduction later in this document.

Learning agent is managed by strategies.tensorforce_trading_strategy to call and configure a tensorforce learning agent.

Tensorforce agent is set up via tensorflow framework

# Running the Program
Two tutorails using sample codes/data in the project will be offered; the runner of the program is set up in strategies.tensorforce_trading_strategies.run. 

Basically, the program iterates on every episodes, and in each episode the program iterates on every timesteps
## inside a timestep:
1. Call agent.act: agent make action(a natural number) based on the state
2. Call environment.execute: bringing action to the trading environment; interpreting the natural number to be a trade; generate reward for the trade (action); generate next state
3. Call agent.observe: judging whether to update networks; recording reward; going into next timestep

# Details of Modules
four modules mentioned in overview, Exchanges | Actions | Reward | Trade will be discussed in this part
## Exchanges
Exchage module is the "market environment" that we normally understand. This module consists of data input of each episode, and obervation generation for each step

I wrote future_exchange.py and future_exchange_position.py based on simulated_exchange.py
### future_exchang.py (for the environment that) (action is trade)
Major changes:
#### Pandas - Numpy
Substitute Pandas procedures with Numpy to improve calculation efficiency
#### exclude_close (new parameter)
If True, the input observation will not include close (price series that is named "close")
#### observe_position (new parameter)
Determines whether the current position (amount of securities hold) will be included into obs and observed by the agent.

This is theoritically very important to make the agent more "sensible"
#### current_price (method)
Separate price series from input data
There will be two separate data generated at each time step:
1. obs: the observations that will be regarded as input to the agent
2. price: the price that will be used to calculate profit and reward
Such separation makes the program more robust to future data issues
#### is_valid_trade (method)
The method to judge whether a trade decision generated by the agent is valid in the market environment
Currently it is defined as:
1. Amount of secureites hold <= 1, and
2. Value of securities hold <= net value of the account
#### reset (method)
Reset will set the Performance(numpy.array that records the running performance including balance, net value, position and price) to initial state (empty).

Before reset, I transform Performance to pandas.dataframe and print the tail of it to the console.

### future_exchange_position.py (for the environment that) (action is position)
This is modified further based on future_exchange.py
Major changes:
#### is_valid_trade (method)
since each action at its timestep is the amount of securities hold, there shouldn't be any restriction on such decision. It will return True everytime.
#### next_price (method)
To judge the action(the amount of securities hold) at this timestep, the program will observe price at the next timestep to give reward.

Notice that this is only used in reward module and the future price will not be passed on to the agent, so there is no future data issues by using this method.

## Action
Action module is responsible to generate the **action space** for the reinforement learning environment, and interpret the **action** (typically a natural number) from the core of agent to a **trade** class

I wrote future_action_strategy.py and future_position_strategy.py based on discrete_action_strategy.py

### future_action_strategy (for the environment that) (action is trade)
By default, each action will consider a trade on 0.1 amount of product, with three choices: buy(long) | neutral | sell (short)

### future_position_strategy (for the environment that) (action is position)
Each action corresponds to the amount of securities hold.

If n_action = 5, there will be five kinds of actions: 0 | 1 | 2 | 3 | 4 that are generated by the agent, which correspond to five kinds of security holdings: -1 | -0.5 | 0 | 0.5 | 1
Major changes:
Methods of last_position and next_price, to 
1. save and read previous security holdings to generate a **trade instance** based on changes
2. read price of next timestep to evaluate the action made at this timestep (coherent with next_price in future_exchange_position.py)

## Reward 
### DirectProfitStrategy (for the environment that) (action is trade)
Notice that this suits only **update by episode** but not **updates by timesteps**
Reward of each episode is calculate by:

security holdings after the action of the last timestep * (price of this timestep - price of the last timestep) - trade fees incured for the action of this timestep

This reward can **not** evaluate the action made at this timestep, but over a whole episode, the sum of rewards equals to the total profit of the agent in this episode, so this is called "DirectProfitStrategy"

### PositionReward (for the environment that) (action is position)
By this file, reward can evaluate the reward of a specific action

reward at each timestep/action = security holdings after this action * (next price - current price) - trade fees (of this action)

## Trade
To support generation and the data flow of trade instance

Trade instances flow among different modules to pass on the specifications of trade action

I added an attribute, *next_price* with default value 0, so that if this attribute is unnecessary the previous programs won't recalibrate

# Case Analysis: PPO(Proximal-Policy-Optimization)
Simple_ppo.py and PPO_learn.py used PPO algorithm that is publicated in 2017 from https://arxiv.org/abs/1707.06347

Both program designed good interfaces to pre-process and pass in data, calibrate hyper-parameters, save/restore agent specifications and performance visualization.

The program is easy and clear to use. Here are some hints that might help you to understand and recall some basic information mentioned above.

The program is composed of several blocks:
## Environment Setup
### Action & Rewards
Choose and call the action and rewards modules for the program.

Here DirectProfitStrategy and FutureActionStrategy should be called and used.

### Feature_pipeline
Data pre-process module that is currently unused but remained in the interface for customized calling.

### Data Input(Exchanges)
Data format: please check Data/TA.csv as an example (futures market data on PTA)

### Agent Specification
Configuration of neural network and hyperparameters
for information on variables, please refer to the original paper of PPO
### Environment Setup
Merge all environment settings together by TradingEnvironment and TensorForceTradingStrategy

## Start Running
simple_PPO.py is the simplest program while PPO_learn configured in-out-sample analysis and save/load of the agent
