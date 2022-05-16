# Quantum Reinforcement Learning

## Requirements:

Code was developed and run on `Python` version `3.9.11`. All main requirements are in [`requirements`](./requirements.txt) file.

## Structure:

```
├───assets                              # assets for readme
├───results                             # results from training
│   ├───classical_QL                    # classical Q-learning results directory
│   ├───classical_DQL                   # classical DQL results directory
│   ├───classical_DQL_sim_quantum       # classical DQL simulating quantum model results directory
│   └───...                             # quantum model results directory
│
├───scripts                             # scripts for generating results
│   ├───src                             # source code directory
│   ├───QML                             # quantum model directory
│   │   1._Classical_QL.py              # classical Q-learning
│   │   2._Classical_DQL.py             # classical Deep Q-learning
│   │   3._Classical_DQL_sim_quant.py   # classical Deep Q-learning simulating quantum circuit
│   │   3b._Classical_DQL_sim_quant_grid_search.py   # script no. 3 for grid search finetuning
│   │   3c._Classical_DQL_sim_quant_finetuning.ipynb # script no. 3 with automatic hyperparameter finetuning
│   
└───tutorials                           # supplementary tutorials to start with

```


## Introduction

The concept of entropy is widely used in various machine learning methods. In this project we wanted to see, if this value behaves in the same way as an analogous quantity used in quantum physics - the entanglement entropy. To investigate this, we used both classical deep Q-learning and its quantum counterpart to train an agent to move in a simple Frozen Lake environment.

The output of a quantum circuit is of course a quantum state, for which the concept of entanglement entropy is pretty straightforward. In contrast, it's not as obvious in the case of classical RL. In that case, we were treating the output generated by the neural network as a quantum state (which in fact, is just some vector).

## Classical method

### Reinforcement learning

Q-learning is a machine learning algorithm of the "Reinforcement Learning" type. The family of such algorithms differs from supervised and unsupervised learning in that the information for training is collected not from data, but from the interaction of the **agent** (trained algorithm) with the **environment**. The function that the agent is guided by is called **politics** and takes the **observation** as an input (e.g. the position of a player on the board, car speed on the track) and returns an **action** (e.g. move right, add gas).

Further reading: https://en.wikipedia.org/wiki/Reinforcement_learning

### Classical Q-Learning

<br/>

>Run in [this script](./scripts/1._Classical_QL.py) 
🚀

<br/>

We will explain the concept of Q-learning on the example of an agent moving around the "FrozenLake" environment. Let's take a 4x4 element board:

```Python
import gym

lake = gym.make('FrozenLake-v1', is_slippery=False)
lake.reset()
lake.render()
```

SFFF

FHFH

FFFH

HFFG

**Legend:**
- S: starting point, one per board,
- F: frozen surface - safe field,
- H: hole - an ice hole, a field that gives a large penalty, or ends the game,
- G: goal, end point.

The agent is always in one of the spaces and, based on his policy, decides to move in one direction (left, right, up, down). The entire walk focuses on the reward, which the agent increases or decreases by entering a specific field.
Scoring example:

- Frozen surface: -0.01 to the reward (penalty to eliminate detour or walking in circles),
- Starting point: -0.01 to reward,
- Hole: -1 to the reward,
- End point: +1 to the reward.

Politics is a Q function that takes an action (e.g. move left) and a state (field 5 - second row from the top, second column from the right) as an input, and returns an appropriate reward. In the the most basic case (without the use of deep learning), the update of the politics (after each step) is represented by the Bellman equation:


![](assets/bellman_equation.png "Bellman equation")


Further reading: https://en.wikipedia.org/wiki/Q-learning



### Deep Q-Learning
<br/>

>Run in [this script](./scripts/2._Classical_DQL.py) 
🚀

<br/>

However, in our case, we use the so-called Deep Q-Learning (DQL). To make decisions, we use a neural network with a 16-element input (number of fields on the board) and a 4-element output, corresponding to the weights provided for each movement:

- 0 - move left,
- 1 - move down,
- 2 - move right,
- 3 - move up.

For example, if we would like to check which way is the best to go from the field number 5, we activate the neuron with index 4

![](assets/dql.png "DQL")


We can access the exact values returned by the neural network as follows

```Python
agent.Qstate(4)
```
tensor([0.4165, 0.6063, 0.5308, 0.4209])


We can see, that the second element is the greatest in value, so the agent is going to move downwards.


### Implementation

The whole thing was coded in pytorch. The architecture of our model is a network consisting of linear layers, followed by a sigmoid activation function:
- https://pytorch.org/docs/stable/generated/torch.nn.Sigmoid.html
- https://pytorch.org/docs/stable/generated/torch.nn.Linear.html

The loss function used was the SmoothL1Loss: https://pytorch.org/docs/stable/generated/torch.nn.SmoothL1Loss.html

Several hundred epochs are enough to train a model. During the training we are tracking the percentage of cases, in which the walk ended up succesfully. For example, if the ratio of walks (over the last 50 epochs) during which the agent ended up in the goal field is equivalent to 90%, we are finishing the training process.


Below we present the average success rate over the last 50 epochs of training, during the total of 1000 epochs:

```Python
plt.figure(figsize=(12,5))
plt.plot(bin_stat.statistic)
plt.title(label=f'Percentage of succesfully finised walks (over the last {epoch_bin} epochs)')
plt.ylim(-1, 101)
plt.xlim(-1, (epochs+1)/epoch_bin)
plt.yticks(ticks=list(range(0, 110, 10)))
plt.xticks(ticks=bin_stat.bin_edges/epoch_bin-1, labels=bin_stat.bin_edges.astype(int))
plt.grid()
plt.show()
```

![](assets/percentage_of_succesfull_walks.png "Percentage of succesfull walks")


Additionally, we check whether the process is optimal or not. We strive for an algorithm that covers the route in 6 steps. If there is more of them, the algorithm is walking in circles. We are looking for the moment where it starts to oscillate around the optimal value. Therefore, we track how many steps an agent takes thanks to its politics, after each epoch:


```Python
plt.figure(figsize=(12,5))
plt.plot(t.jList)
plt.yticks(ticks=list(range(0, 110, 10)))
plt.title(label="Number of states done by the agent")
plt.grid()
plt.show()
```

![](assets/number_of_steps.png "Number of steps")

Let's see how the trained agent works:

![](assets/agent_walk.gif "Agent walk")


## Variational Quantum Circuit
<br/>

>Run from [this folder](./scripts/RUN_QML/) 
🚀

<br/>

In the quantum approach we are replacing the neural network with the so-called Variational Quantum Circuit (VQC). This is a type of quantum circuit with manipulable (classical) parameters. Like neural networks, VQCs can approximate arbitrary functions or classifiers. The following implementation is taken from

[S. Y.-C. Chen, C.-H. H. Yang, J. Qi, P.-Y. Chen, X. Ma and H.-S. Goan, *"Variational Quantum Circuits for Deep Reinforcement Learning"*, 2019](https://arxiv.org/abs/1907.00397)

### Encoding of the state

Recall that our task is, given a state (the place on the board where our agent is), to find the best corresponding move. The board contains 16 possible states (0-15), which can be encoded using 4 bits (0000-1111)

<!-- $$ b_1 b_2 b_3 b_4, $$ -->
![equation](https://latex.codecogs.com/svg.image?b_1&space;b_2&space;b_3&space;b_4,)

or similarly 4 qubits

<!-- $$ |b_1\rangle \otimes |b_2\rangle \otimes |b_3\rangle \otimes |b_4\rangle. $$ -->
![equation](https://latex.codecogs.com/svg.image?|b_1\rangle&space;\otimes&space;|b_2\rangle&space;\otimes&space;|b_3\rangle&space;\otimes&space;|b_4\rangle.)

For example, if we are in the 13th state, its bitwise representation is 1101, which can be also written down using qubits' states as ![equation](https://latex.codecogs.com/svg.image?|1\rangle&space;\otimes&space;|1\rangle&space;\otimes&space;|0\rangle&space;\otimes&space;|1\rangle). 

Finally, the explicit gates which must be applied on consecutive qubits to encode agent's state are shown below:

![](assets/encoding.jpg "Encoding of agent's state")

Note, that we are using the ![equation](https://latex.codecogs.com/svg.image?\theta_i) and ![equation](https://latex.codecogs.com/svg.image?\phi_i) parameters. These parameters won't be trained in the further part of the procedure, and are only used to encode the state properly.

### Layers

On the encoded state we are acting with the following gates:

![](assets/layer_1.jpg)

At the beginning we are entangling all of the qubits using the *CNOT* gates. Then, we are rotating each qubit along the *X*, *Y* and *Z* axes according to the following formula

<!-- $$ R(\alpha_i, \beta_i, \gamma_i) = R_x(\alpha_i) R_y(\beta_i) R_z(\gamma_i) $$ -->
![equation](https://latex.codecogs.com/svg.image?R(\alpha_i,&space;\beta_i,&space;\gamma_i)&space;=&space;R_x(\alpha_i)&space;R_y(\beta_i)&space;R_z(\gamma_i))

![equation](https://latex.codecogs.com/svg.image?\alpha), ![equation](https://latex.codecogs.com/svg.image?\beta) and ![equation](https://latex.codecogs.com/svg.image?\gamma) are the parameters that will be optimized during each training's iteration (which also means, that the gradient will be calculated exactly over these variables).

At the very end we are conducting a measurement of each qubit and basing on the output we make a proper move. The whole circuit looks as follows:

![](assets/circuit.jpg "The whole VQC")

### Early stopping

Because the training proces of VQC is very unstable we are (similarly as in the case of classical approach) using early stopping. We are terminating the procedure if during the last 20 epochs the reward didn't change and was positive.

## Extended classical model
<br/>

>Run in [this script](./scripts/3._Classical_DQL_sim_quant.py) 
🚀

<br/>


It's important to note, that the state vector given by the VQC consists of ![equation](https://latex.codecogs.com/svg.image?2^n) complex variables, where *n* is the number of qubits, which in our case gives 16 numbers. To resemble this situation as close as possible we are extending the classical model by increasing the number of neurons in each layer (also the output one) to 32. We are using 32 real numbers to encode 16 complex ones.

It should be noted, that unfortunately due to this extension the agent's training becomes more unstable and converges more slowly.


## Results
### Quantum DQ-learning

 With earlystopping | WithOUT earlystopping 
 :---: | :---: 
![earlystopping_reward](results/quantum/earlystopping_Quantum_DQN_Frozen_Lake_NonSlip_Dynamic_Epsilon_RMSProp_REWARD_NO20220425223058.png) | ![without-earlystopping_reward](/results/quantum/Quantum_DQN_Frozen_Lake_NonSlip_Dynamic_Epsilon_RMSProp_REWARD_NO20220426183607.png)
![earlystopping_entropies](results/quantum/earlystopping_entropies.png) | ![without-earlystopping_entropies](/results/quantum/entropies.png)

[comment]: # (Te opisy before loss, after loss mogą być mylące, a i tak nie wnoszą za dużo)

As we can see, two kinds of entropy behave differently, although there is some correlation between them.

Decrease in entropy often correlates with better reward. It is best seen near the end of the learning process, so entropy reduction might signal a stabilization of the learning process. But it doesn't apply the other way (lower entropy does not correlate with steady reward).

### Classical Q-learnig:

Classical Q-learning model learns for wide range of parameters and has very predictible behaviour: 
* 1. agent wanders untils spot reward 
* 2. then stays on that path usually optimizing path to the shortest one in few epochs
The only fun is to set hyperparamers to converge as fast as possible. 

![image](./assets/QL_results.jpg)

We didn't calculate here entanglement entropies. This one is just complementary proof of concept.

### Deep Q-learning:

Learns slower than non-deep version and is slightly more sensitive to hyperparameters. Almost always converge to optimal number of steps but sometimes vary a lot along the way:

![image](./assets/DQL_results.jpg)

Here we've calculated entropy after every step (in contrary to every epoch for other values):

![image](./results/classical_DQL/entropies.jpg)

It's behaviour is very random, but in the end Shannon entropy stops around `0` and entanglement entropy reach almost `2`. It only occurs, when the model train good path (converge to winning strategy).

### Deep Q-learning simulating quantum circuit (Extended classical model):

Here was a lot of trouble to train the model. We were seeking architecture and hyperparameters in 3 stages:

### **Finetuning stages:**
-----

#### **1. Manual finetuning on baseline model:**

* *Model*: one/two hidden layer and sigmoid activation function
* *Goal*: We perform a few experiments to gain a sense how and which hyperparamers influence our model training. Also we had baseline ranges to start more systematic searching
* *Results*: It turns out the model is very hard to train, in fact only for some very narrow ranges we obtained ~40% win ratio in last few epoch means. It occur rarely, only for some seeds. 

User can run our [script](./scripts/3._Classical_DQL_sim_quant.py), all paramers there are the results of this finetuning. 

#### **2. Grid search for the best architecture:**

* *Model*: 
    * **hidden layers**: from 1 to 6 incl.
    * **Activation functions**: sigmoid, hyperbolic tangent and leaky relu with slope 0.1
* *Goal*: Here we put training with slightly lower hyperparameters, which gaves training 'pace' (learning rate, random paramer scaling etc.) and run every combination of tested architectures for `20'000` epochs to choose best architecture to hyperparameter finetuning.
* *Results*: 
    * None of the models trained to win. 
    * Most promising results shown leaky relu, but we quit it in next stage since it 'favors' positive values.
    * Tangent performs the worst.
    * Model starts to train for 1 and 2 hidden layers, for 3 and above hidden layers the architecture seems to complicated.

All the results are in [results directory](./results/classical_DQL_sim_quantum/). Script used for training is [here](./scripts/3b._Classical_DQL_sim_quant_grid_search.py).

#### **3. Automatic hyperparamers finetuning :**

For this stage we user `pyTorch` [finetuning tutorial](https://pytorch.org/tutorials/beginner/hyperparameter_tuning_tutorial.html) with `ray`.

* *Model*: 
    * **hidden layers**: from 1 to 2 incl.
    * **Activation functions**: sigmoid
* *Goal*: Final, automatic, full scale finetuning. 
* *Results*: 
    * ... (in progress) ...

Notebook used for training is [here](./scripts/3c._Classical_DQL_sim_quant_finetuning.ipynb). Finetuning was performed on Google Collab on P100 GPU with 20 parallel experiments.

### Final results:

For best model which almost trained, we can see convergence of (both) entropies to zero, right in epochs, where model started to reach goal:

![image](./assets/DQL_sim_quant_results.jpg)
![image](./results/classical_DQL_sim_quantum/_BEST_1_layers_sigmoid_activation/entropies.jpg)

All parameters are in [results folder](./results/classical_DQL_sim_quantum/_BEST_1_layers_sigmoid_activation/). 

However this method is incomparably harder to obtain effective model for environment. Not only in terms of hyperparameters sensitivity, but also from training duration and only small amount of 'succesful' experiments i.e. model which has mean win ratio ~40%. Also to obtain 'succesful' model we need to stop in right place i.e. for smaller win ratio early stop condition, which does not guarantee optimal path.

Interesting is, that models with 'real' quantum circuits (not simulated by neural network) were able to train, even if rarely. 
This shows that simulating quantum distributions for classical nerual networks can be tough. In our case particularly with:
* classical data encoded with basis embedding 
* classical data decoded with expectation values from Pauli Z operator (with flipped sign)

Similar conclusions, that quantum disribution can be hard to simulate, can be found in literature e.g. *Learning hard quantum distributions with variational
autoencoders* [Rochetto et, al. 2018](https://www.nature.com/articles/s41534-018-0077-z.pdf).
