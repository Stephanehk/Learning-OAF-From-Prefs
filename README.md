# Learning Optimal Advantage from Preferences and Mistaking it for Reward
This is the official implementation for the paper Learning Optimal Advantage from Preferences and Mistaking it for Reward.

## Setup

Please use ```pip install -r requirements.txt``` to install all necessary libraries.

## Mapping from paper to code

Below we describe where to find each of the algorithm components as depicted in Figure 1.

<p align="center">
  <img src="https://i.imgur.com/lygbALW.png" />
  Figure 1
</p>


### Dataset created by reward function r
A more detailed description of the dataset itself can be found in the section below. The function responsible for generating synthetic preferences labeled by either the regret or partial return model is located in ```learn_advantage.algorithms.advantage_learning.generate_synthetic_prefs```. For this submission, all preferences are generated by the regret preference model (indicated by the ```preference_model = "regret"``` flag) in accordance with Eq. 3 of the paper.

Note that ```learn_advantage.algorithms.advantage_learning.generate_synthetic_prefs``` is called from ```learn_advantage.advantage_learning_exp_handler.py```, which loads the dataset and randomly samples segment pairs for labeling. A new random seed is used every time this is done. After ```learn_advantage.algorithms.advantage_learning.generate_synthetic_prefs``` is executed, ```learn_advantage.algorithms.advantage_learning.augment_data``` is called. This function doubles the preference dataset by copying each segment pair and then swapping the segments and corresponding preference labels.

### Algorithm for learning from preferences
The first/third algorithm learns g directly from the preference dataset. This algorithm is instantiated via the ```learn_advantage.algorithms.models.RewardFunctionPR```class.  ```learn_advantage.algorithms.models.RewardFunctionPR.forward``` takes in a dataset of segment pair features and returns the preference probabilities for each segment pair under the model's current weight vector (Eq. 6 of the paper). The model parameters are then updated using the true preference labels in ```learn_advantage.algorithms.advantage_learning.run_single_set``` with the loss function defined in ```learn_advantage.algorithms.advantage_learning.reward_pred_loss```.

The second algorithm learns with regret assumption. This algorithm is instantiated via the ```learn_advantage.algorithms.models.RewardFunctionRegret```class. The same process as above is then followed.

Note that the instantiation of each of these models, as well as the training loop where ```learn_advantage.algorithms.advantage_learning.run_single_set``` is called, can be found in ```learn_advantage.algorithms.advantage_learning.train```.

### Output of learning from preferences
Upon completion, ```learn_advantage.algorithms.advantage_learning.train``` returns the learned weight vector, the training losses, and the GPU+CPU training time if a GPU was used. Note that if the regret model was used for learning then the weight vector that achieved the lowest loss is returned instead of the most recently updated weight vector. The weight vector is then saved in  ```learn_advantage.advantage_learning_exp_handler.py``` to the directory specified by command line arguments.

### Additional step to create policy (and evaluate that policy)
The ```learn_advantage.analysis.get_scaled_returns.py``` files handles loading in the learned weight vectors and evaluating them. This file serves as a wrapper for the ```learn_advantage.utils.utils.get_scaled_return``` function, which first takes in a learned weight vector and derives a policy from this vector. If the learned weight vector is to be treated as an optimal advantage function (as indicated by the ```learn_oaf = True``` flag), then ```learn_advantage.algorithms.rl_algos.build_pi_from_feats``` is called. This function returns a policy that always acts greedily over the given weight vector. Otherwise, if the learned weight vector is to be treated as a reward function, then ```learn_advantage.algorithms.rl_algos.value_iteration``` is called in order to learn the Q-function for this reward function. A policy that acts greedily over this Q function is then returned using the ```learn_advantage.algorithms.rl_algos.build_pi``` function.

In ```learn_advantage.utils.utils.get_scaled_return```, the returned policy, as well as the ground truth reward function, is then passed to ```learn_advantage.algorithms.rl_algos.iterative_policy_evaluation```. This function performs policy evaluation on the given policy under the ground truth reward function. In ```learn_advantage.utils.utils.get_scaled_return``` the mean scaled return is then computed using this result as well as the average return of an optimal policy and a uniformly random policy. The mean scaled return serves as the backbone of our analysis, where an optimal policy has a mean scaled return of 1, a uniformly random policy has a mean scaled return of 0, and a worse-than-random policy has a mean scaled return of less than 0. We considered a mean scaled return of > 0.9 to be near-optimal.

## Dataset
Our dataset consists of randomly generated MDPs, each accompanied by a dataset of synthetic preferences labeled by the regret preference model. For our results in this paper, we use a set of 190 randomly generated MDPs.

The results in Figures 3, 4, 10, and 11 are across 30 randomly generated MDPs, which are located in ```data/input/random_MDPs/```. Results for Figure 6 are across 100 MDPs. If you want to reproduce that figure, please download/unzip the ```random_mdps_150-200.zip``` file from our CMT submission and unzip the contents into the ```random_MDPs/``` provided in this directory. Otherwise, you may run the same code used to generate Figure 6 with a smaller subset of the MDPs.

The results in Figure 5 are across 90 randomly generated MDPs. Because of the file size restrictions, for this submission, we have only provided a representative sample of 30 MDPs. This sample contains 10 MDPs from each of the 3 conditions described in Appendix B, and are located in the ```data/input/fig_5_random_MDPs``` directory.

Note that the script used to generate all these MDPs is entitled ```learn_advantage.env.generate_random_mdp.py``` and provided in this repository.

## Reproducing/replicating results from our submission
To run the experiments from Figures 3, 4, 10, and 11, run the script ```scripts/run_no_absorbing_transitions_exp.sh``` from the project root directory. The following will be printed out:
- The scaled returns achieved by each model for each experimental condition for each MDP (plotted in Figures 3/11).
  - Note: A scaled return of 1 indicates optimal performance, 0 indicates uniformly random performance, and < 0 indicates worse than random
    performance.
- The mean maximum learned advantage for each state for each experimental condition for each MDP (plotted in Figures 4/10).
- The statistics returned by a Wilcoxon signed rank test, indicating the significance of the above two results.

To run the experiments from Figure 5, run the script ```scripts/test_loop_hypothesis.sh``` from the project root directory. The following results will be returned:
- The number of points that agree with our hypothesis presented in Section 3.3.
- A graph similar to that from Figure 5, visually depicting the number of points that agree with our hypothesis presented in Section 3.3.

To run the experiments from Figure 6, run the script ```scripts/test_shaped_reward_assumption.sh``` from the project root directory. The following result will be returned:
- A graph similar to that from Figure 6, visually depicting training curves when running Q learning and treating the learned optimal advantage function as a reward function versus using the true reward function.

## Running less computationally intensive but representative results
The ```scripts/test_shaped_reward_assumption.sh``` and ```scripts/run_no_absorbing_transitions_exp.sh``` scripts may be particularly computationally expensive, taking a few days on a GPU to run. We provide the ```TEST``` flag (used as follows: ```./scripts/test_shaped_reward_assumption.sh TEST``` or ```./scripts/run_no_absorbing_transitions_exp.sh TEST```), which runs the same experiments presented in the paper but across a smaller dataset of MDPs and with smaller preference datasets. These scripts should take only a few hours to run on a CPU. The outputted results should illustrate the general patterns described in the paper, such as achieving better performance with the Greedy A*_hat model or the maximum learned advantages moving closer to 0 when training with transitions from the absorbing state. Note however that there will be some more variance and deviations from our reported results because of the small dataset sizes.

Note that you may change the following in order to see representative results faster:
- Instead of using preference dataset sizes of [300,1000,3000,10000,30000,100000], try only using a single preference data size by setting the ```--num_prefs``` argument.
- Instead of running experiments across 30-100 MDPs, try using fewer MDPs by setting the ```--end_MDP``` argument. Note that the first MDP (ie: ```--start_MDP```) starts at index 100 so ```--end_MDP``` must be greater than 100.
- Instead of training for 1,000+ epochs, try training for only a few hundred epochs by setting the ```--N_ITERS``` argument.
