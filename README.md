# Extended Kalman Filter

Tracking an objects state based on knowledge of its system dynamics and the 
availability of noisy measurements.

The goal of this project ist to design an **Extended Kalman Filter** that is able
to track a bicycle driving around a car. The Kalman Filter will continuously 
predict the bicycles state using the bicycles system dynamics and correct the 
predicted state by the measurements provided by the cars lidar and radar sensors.

In the video below the bicycle is represented by the blue car. The blue markers 
illustrate the radar measurements, the red ones the lidar measurements and the green 
triangles represent the bicycle state estimations generated by the Kalman filter. 

![][intro]

We will subdivide the Kalman Filter implementation into two main parts:

- in a first step we will define the required **system equations**, namely the bicycle 
**motion model** and the lidar and radar **measurement models**
- subsequently we will implement these models into the Extended **Kalman Filter 
process equations**

The Kalman Filter C++ code can be found in the ``src`` folder. It can be run 
following [these instructions][starter].

# Dynamic system equations

## The bicycle - A linear motion model

| motion model | process noise | 	
| ---------- | -------------|  	
| ![][motion_model] | ![][process_noise] |

The bicycle is represented by a simple linear motion model with 

- a state ***x*** tracking the ***p***ositions and ***v***elocities in **x**- and 
**y**-directions for the time steps ***k***
- the transition matrix ***F*** propagating the previous state to the new predicted 
state using the bicycles kinematics
- a possible control term represented by the control matrix ***B*** and the control 
vector ***u*** that may cover known external influences like the acceleration 
when pushing a cars throttle. However, we will omit this term for the bicycle model 
as we have no information on how it is accelerating. Instead we will incorporate the 
acceleration into the process noise
- the process noise ***v*** is drawn from a normal distribution with a zero-mean and 
an uncorrelated process covariance ***Q***. The noise covers unknown external 
influences like the wind, road bumps or in our case the bicycles acceleration

Let's have a closer look on the motion model parameters. We will highlight 
the parts in red that need to be integrated into the Kalman filters C++ code.
 
Each new state is predicted by the motion model using the previous state and the 
bicycle kinematics as follows 

![][motion_model_parameters]

The acceleration parts are pulled out and incorporated into the process noise. 
The noise vector can be divided into the matrix ***G*** that does not contain any 
random variables and the vector ***a*** that contains the unknown accelerations.

![][motion_model_noise]

The process [covariance][covariance_definition] is defined by 
![][process_covariance_def]

Since the covariance is assumed to be uncorrelated, the correlated terms are set to 
zero. Performing the matrix multiplication results in 

![][process_covariance]

The process covariance is updated at each new time step. Since the accelerations 
are unknown, longer time steps result in higher uncertainty as the bicycle will
change its state in an unpredictable manner.

The values for the acceleration variances are provided as input for this project and 
are defined as follows

![][process_variance]

This completes the motion model definition. Now let us have a look on the second model 
required by the Kalman filter: the measurement model. Since our car is tracking the 
bicycle with both its lidar and radar sensors we will define two measurement models.

## Lidar - A linear measurement model
| LIDAR measurement model | LIDAR noise | 	
| ---------- | -------------|  	
| ![][lidar_model] | ![][lidar_noise] |

The measurement model describes how the sensor observes the true state of the bicycle.
This observation must not necessarily be performed in the same units or scale as 
the state. However, for the lidar this is partly the case since 

- the lidar is able to measure the **x** and **y** positions of the bicycles state. 
- No sensor is 100% precise. The sensors uncertainty is represented 
by the measurement noise ***w***.
- As for the motion model, the noise is drawn from a normal distribution 
with zero-mean and an uncorrelated covariance ***R***.
- The observation or sensor matrix ***H*** maps the bicycles state into the 
measurement space. In other words, the sensor matrix transforms the state into the format
in which the sensor observes the true state. In the case of the lidar this only means 
that the velocity components of the bicycle state are being dropped, since the sensor 
only measures the bicycles location.

The measurement model is linear since the mapping can be performed by a simple 
matrix-vector multiplication.

![][lidar_parameters]

The sensor covariance is usually provided by the manufacturer and is defined in 
this project as follows

![][lidar_covariance] 


## Radar - A nonlinear measurement model

![][radar_mapping]

Unlike the lidar model, the radar model is nonlinear. This is due to
the fact that it measures the bicycles state in polar coordinates. 

| Radar measurement model | Radar noise | 	
| ---------- | -------------|  	
| ![][radar_model] | ![][radar_noise] |

The mapping of the cartesian state into the polar sensor space is not possible by 
a simple linear transformation. Some nonlinear trigonometric transformations are 
required which are performed by the observation model ***h***. 

![][radar_mapping_equation]

A detailed derivation of this function is shown in 
[Mercedes' EKF references][ekf_references].

![][radar_parameters]

Finally the radar covariance is provided as follows

![][radar_covariance]

# The Kalman filter process
Having defined the motion and measurement models, let's have a look on the 
Kalman filter process. It can be divided into the following parts:

1. Initialization of the state and its covariance
2. State prediction at the current time step using the motion model
3. Determination of the Kalman gain and the innovation for the incoming measurement 
4. Update of the predicted state by incorporating the measurement information
5. New cycle for the next time step

![][kalman_filter]

## 1. State initialization
Before entering the Kalman filter cycle, the bicycles state ***x*** and its 
covariance ***P*** need to be initialized. The first incoming measurement will 
be used for this task. If it is a lidar measurement it can directly be used to 
initialize the states location. The radar measurement needs to be converted from 
polar into cartesian coordinates. In both cases the velocity is initialized to zero.

| Initialization by lidar measurement | Initialization by radar measurement | 	
| :----------: | :-------------: |
| ![][init_lidar] | ![][init_radar] |  

Since we are more confident on the bicycles initial position than on its velocity
we will initialize the state covariance in accordance to our uncertainty: low 
variances for the positions and high variances for the velocities. The correlated 
variances are initialized to be zero.

![][init_covariance]

## 2. Prediction
| motion model | process noise | 	
| ---------- | -------------|  	
| ![][motion_model_uncommented] | ![][process_noise_uncommented] | 

Following the initialization, a prediction of the new bicycle state is performed 
using the motion model. The predicted state is defined as the estimate of the true 
state. Note that we omitted the control term for our bicycle model. 
Furthermore, since the process noise has a zero-mean, its estimated value is zero.

![][pred_state_est]

The predicted state is also called the ***a priori*** state since the measurement 
information is not yet included. *A priori* states are indicated by the minus 
superscript. After having updated the state prediction with the measurement 
information we get the ***a posteriori*** state, indicated by the plus superscript.
The little hat "**^**" over the state signifies that it is an estimated state.
The *a priori* state of the current time step is predicted using the *a posteriori* 
state of the previous time step. 

Since the acceleration of the bicycle is unknown, we cannot be 100% sure about where 
the bicycle state will be after the time step. This uncertainty is represented by the 
state covariance ***P***. The *a priori* state covariance is defined as the estimate 
of the state estimation error (the difference between the true state and the 
predicted state):

![][pred_cov_est]

The increase of uncertainty is mathematically realized by adding the process 
covariance ***Q*** of the current time step. We remember, that the process covariance
contains the acceleration dependent variances.

## 3. Determination of the innovation and the Kalman gain 
 
Following the prediction, a laser or radar measurement may come in. This measurement
is used to determine the innovation ***y***. The innovation is defined as the 
difference between the measurement and the predicted state. It can be interpreted as the 
part of the measurement that brings in new information about the state. 
The state needs to be mapped into the measurement space by the sensor function ***h*** 
which was introduced in the system equations section above.

| innovation | lidar model | radar model | 	
| :----------: | :----------: | :----------: |  	
| ![][innovation] | ![][h_lidar] | ![][h_radar] |

The second parameter determined in this step is the Kalman gain ***K***. The 
Kalman gain combines the uncertainty of the state ***P*** with the uncertainty 
of the measurement ***R***. It can be interpreted as a weighting factor, deciding how
much of the innovation is used to update the predicted state based on how much it
trusts the measurement or the current state prediction. We will come back to this in 
the next section.

For linear measurement models - like in the case of the lidar - the Kalman gain can be 
computed using the sensors observation matrix ***H*** to map the state covariance 
into the measurement space.

![][kalman_gain]

However this is not valid for nonlinear measurement models - as in the case 
of the radar. This is due to the fact that gaussian distributions being mapped 
by a nonlinear function are are not gaussian anymore. This is shown in the following 
illustration 

| linear model | nonlinear model | linearized model | 	
| :----------: | :----------: | :----------: |
| ![][linear_model] | ![][nonlinear_model] | ![][linearized_model]

Source: [Robot Mapping Extended Kalman Filter][EKF]

Here is where the **Extended Kalman Filter (EKF)** comes into play. In order to be 
able to use the nonlinear radar model it needs to be linearized. We remember that the 
radar model is defined as follows 

![][radar_model_linearization]

The EKF performs the linearization by a first order [Taylor Expansion][Taylor] using 
the most recent predicted state as operating point:

![][linearization]

The determination of the Jacobian matrices ***Hj*** and ***Mj*** is necessary for the 
linearization. The derivation requires some analysis and can again be found in 
[Mercedes' EKF references][ekf_references].

| Measurement model jacobians |	
| :-------------------- |
| ![][Hj] 
| ![][Mj]

The Kalman gain for the radar sensor can then be determined by swapping the
linear observation matrix with the jacobian

![][nonlinear_gain]


## 4. Measurement update
Having determined the innovation ***y*** and the Kalman gain ***K*** the measurement
information can be used to update the predicted *a priori* state and determining the 
*a posteriori* state.

![][update_state]

The *a priori* state covariance needs also to be updated. For linear measurement models 
as the lidar this can be performed using the sensor matrix ***H*** as follows

![][update_covariance_linear]

However, for the nonlinear measurement models like the radar the sensor matrix needs to 
be swapped by the jacobian as when computing the Kalman gain

![][update_covariance_nonlinear]

## 5. Cycle
Having determined the *a posteriori* state and its covariance a new cycle is started 
by turning the current state into the previous state and making a new prediction on it.

## Intuition on the Kalman gain
Previously we mentioned that the Kalman gain can be interpreted as a weighting factor, 
deciding how much of the innovation is used to update the predicted state based on how 
much it trusts the measurement or the current state prediction.  

| Kalman gain (lidar) | Update step |
| :----------: | :----------: |
| ![][kalman_gain_black] | ![][update_state_black]

Let's have a closer look on three edge cases to get an intuition on how the Kalman gain 
works.

#### A) Little trust into the measurement
If we have little trust into the measurement its covariance ***R*** is high and the 
Kalman gain tends towards

![][kalman_gain_rinf]

In that cases the Kalman filter ignores large parts of the innovation. The updated state 
will be close to the predicted state, irrespective of the obtained measurement 
information.

![][update_state_k0]

#### B) High trust into the measurement
If we have high trust into the measurement its covariance ***R*** is low and the 
Kalman gain tends towards

![][kalman_gain_r0]

In that cases the Kalman filter ignores large parts of state prediction. The updated state 
will be close to the measured state.

![][update_state_kh-1]


#### C) High trust into the predicted state
If we have high trust into the state prediction its covariance ***P*** is low and the 
Kalman gain tends towards

![][kalman_gain_p0]

In that cases the Kalman filter ignores large parts of the innovation. The updated state 
will be close to the predicted state, irrespective of the obtained measurement 
information.

![][update_state_k0]

This is not always desired, since we may lose valuable measurement information in 
that cases. However, the process noise respectively the process covariance ***Q*** 
helps in preventing the estimated state covariance ***P*** to become to small. 

![][pred_cov_black]

Note that the Kalman Filter requires the consideration of uncertainty to become 
more robust.   

# Project Results
The following videos show the results when applying the Extended Kalman filter on the 
two provided datasets of the [Term 2 Simulator][simulator]. In both cases the 
estimated state compared to the ground truth meets the requirement of a RMSE
below or equal to the values ``[.11, .11, 0.52, 0.52]`` after some "warm-up" time. 


| Dataset 1 | Dataset 2 |
| :----------: | :-------------: |  	
| ``videos/dataset_1_close.mp4`` | ``videos/dataset_2_far.mp4`` |
| ![][close1] | ![][far2] |


[intro]: ./videos/intro.gif "Intro"
[starter]: ./EKF_starter.md "EKF Starter Code"
[motion_model]: ./images/motion_model.png "motion model"
[process_noise]: ./images/process_noise.png "proces noise"
[motion_model_parameters]: ./images/motion_model_parameters.png "motion model parameters"
[motion_model_noise]: ./images/motion_model_noise.png "motion model noise"
[process_covariance_def]: ./images/process_covariance_definition.png "process covariance"
[process_covariance]: ./images/process_covariance.png "process covariance"
[process_variance]: ./images/process_variance.png "process variance"
[covariance_definition]: https://en.wikipedia.org/wiki/Covariance#Definition
"https://en.wikipedia.org/wiki/Covariance#Definition"
[lidar_model]: ./images/measurement_model_lidar.png "lidar measurement model"
[lidar_noise]: ./images/measurement_noise_lidar.png "lidar noise"
[lidar_parameters]: ./images/lidar_parameters.png "lidar parameters"
[lidar_covariance]: ./images/lidar_covariance.png "lidar covariance"
[radar_model]: ./images/measurement_model_radar.png "radar measurement model"
[radar_noise]: ./images/measurement_noise_radar.png "radar noise"
[radar_mapping]: ./images/radar_mapping.png "radar mapping"
[radar_mapping_equation]: ./images/radar_mapping_equation.png "radar mapping"
[radar_parameters]: ./images/radar_parameters.png "radar parameters"
[ekf_references]: ./Docs/sensor-fusion-ekf-reference.pdf "./Docs/sensor-fusion-ekf-reference.pdf"
[radar_covariance]: ./images/radar_covariance.png "radar covariance"
[kalman_filter]: ./images/kalman_filter.png "Kalman filter process" 
[init_radar]: ./images/init_radar.png "initialization by radar" 
[init_lidar]: ./images/init_lidar.png "initialization by lidar" 
[init_covariance]: ./images/init_covariance.png "covariance initializationr"
[motion_model_uncommented]: ./images/motion_model_uncommented.png "motion model"
[process_noise_uncommented]: ./images/process_noise_uncommented.png "proces noise"
[pred_state_est]: ./images/predicted_state_estimate.png "predicted state estimate"
[pred_cov_est]: ./images/predicted_cov_estimate.png "predicted process covariance estimate"
[innovation]: ./images/innovation.png "innovation"
[h_lidar]: ./images/h_lidar.png "lidar model"
[h_radar]: ./images/h_radar.png "radar model"
[kalman_gain]: ./images/kalman_gain.png "Kalman gain lidar"
[linear_model]: ./images/Linear_Model_50.png
[nonlinear_model]: ./images/NonLinear_Model_50.png
[linearized_model]: ./images/Linearized_Model_50.png
[EKF]: http://ais.informatik.uni-freiburg.de/teaching/ws12/mapping/pdf/slam03-ekf.pdf
"http://ais.informatik.uni-freiburg.de/teaching/ws12/mapping/pdf/slam03-ekf.pdf"
[Taylor]: https://en.wikipedia.org/wiki/Taylor_series "https://en.wikipedia.org/wiki/Taylor_series"
[radar_model_linearization]: ./images/radar_model_linearization.png "radar model"
[Simon]: https://onlinelibrary.wiley.com/doi/book/10.1002/0470045345 
"https://onlinelibrary.wiley.com/doi/book/10.1002/0470045345"
[linearization]: ./images/linearization.png "linearization"
[Hj]: ./images/Hj.png "Measurement model Jacobian"
[Mj]: ./images/Mj.png "Process noise Jacobian"
[nonlinear_gain]: ./images/nonlinear_gain.png "Kalman gain radar"
[update_state]: ./images/update_state.png "Update the state"
[update_covariance_linear]: ./images/update_covariance_linear.png "Update the covariance"
[update_covariance_nonlinear]: ./images/update_covariance_nonlinear.png "Update the covariance"
[kalman_gain_black]: ./images/kalman_gain_black.png "Kalman gain lidar"
[pred_cov_black]: ./images/pred_cov_black.png "predicted process covariance estimate"
[update_state_black]: ./images/update_state_black.png "update state"
[pred_cov_black]: ./images/pred_cov_black.png "predicted process covariance estimate"
[kalman_gain_rinf]: ./images/kalman_gain_Rinf.png "Little trust in measurement"
[update_state_k0]: ./images/update_state_K0.png "Little trust in measurement"
[kalman_gain_r0]: ./images/kalman_gain_R0.png "High trust in measurement"
[update_state_kh-1]: ./images/update_state_KH-1.png "High trust in measurement"
[kalman_gain_p0]: ./images/kalman_gain_P0.png "High trust in prediction"
[close1]: ./videos/dataset_1_close.gif "Dataset1 close view"
[far2]: ./videos/dataset_2_far.gif "Dataset2 far view"
[simulator]: https://github.com/udacity/self-driving-car-sim/releases
"https://github.com/udacity/self-driving-car-sim/releases"