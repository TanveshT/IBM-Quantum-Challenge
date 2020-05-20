# Exercise 2: Measurement Error Mitigation

Present day quantum computers are subject to noise of various kinds. The principle behind error mitigation is to reduce the effects from a specific source of error.  Here we will look at mitigating measurement errors, i.e., errors in determining the correct quantum state from measurements performed on qubits.

<img src="images/mitigation.png" width="900"/>
<center>Measurement Error Mitigation</center>

In the above picture, you can see the outcome of applying measurement error mitigation. On the left, the histogram shows results obtained using the device `ibmq_vigo`. The ideal result should have shown 50% counts $00000$ and 50% counts $10101$. Two features are notable here:

- First, notice that the result contains a skew toward $00000$. This is because of energy relaxation of the qubit during the measurement process. The relaxation takes the $\vert1\rangle$ state to the $\vert0\rangle$ state for each qubit.
- Second, notice that the result contains other counts beyond just $00000$ and $10101$. These arise due to various errors. One example of such errors comes from the discrimination after measurement, where the signal obtained from the measurement is identified as either $\vert0\rangle$ or $\vert1\rangle$.

The picture on the right shows the outcome of performing measurement error mitigation on the results. You can see that the device counts are closer to the ideal expectation of $50%$ results in $00000$ and $50%$ results in $10101$, while other counts have been significantly reduced.


## How measurement error mitigation works


We start by creating a set of circuits that prepare and measure each of the $2^n$ basis states, where $n$ is the number of qubits. For example, $n = 2$ qubits would prepare the states $|00\rangle$, $|01\rangle$, $|10\rangle$, and $|11\rangle$ individually and see the resulting outcomes. The outcome statistics are then captured by a matrix $M$, where the element $M_{ij}$ gives the probability to get output state $|i\rangle$ when state $|j\rangle$ was prepared. Even for a state that is in an arbitrary superposition $|\psi \rangle = \sum_j \alpha_j |j\rangle$, the linearity of quantum mechanics allows us to write the noisy output state as $|\psi_{noisy}\rangle = M |\psi\rangle$.

The goal of measurement error mitigation is not to model the noise, but rather to apply a classical correction that undoes the errors. Given a noisy outcome, measurement error mitigation seeks to recover the initial state that led to that outcome. Using linear algebra we can see that given a noisy outcome $|\psi_{noisy}\rangle$, this can be done by applying the inverse of the matrix $M$, i.e., $|\psi \rangle = M^{-1} |\psi_{noisy}\rangle$.  Note that the matrix $M$ recovered from the measurements is usually non-invertible, thus requiring a generalized inverse method to solve.  Additionally, the noise is not deterministic, and has fluctuations, so this will in general not give you the ideal noise-free state, but it should bring you closer to it.

You can find a more detailed description of measurement error mitigation in [Chapter 5.2](https://qiskit.org/textbook/ch-quantum-hardware/measurement-error-mitigation.html) of the Qiskit textbook.

**The goal of this exercise is to create a calibration matrix $M$ that you can apply to noisy results (provided by us) to infer the noise-free results.**

---
For useful tips to complete this exercise as well as pointers for communicating with other participants and asking questions, please take a look at the following [repository](https://github.com/qiskit-community/may4_challenge_exercises). You will also find a copy of these exercises, so feel free to edit and experiment with these notebooks.

---

In Qiskit, creating the circuits that test all basis states to replace the entries for the matrix is done by the following code:


```python
#initialization
%matplotlib inline

# Importing standard Qiskit libraries and configuring account
from qiskit import IBMQ
from qiskit.compiler import transpile, assemble
from qiskit.providers.ibmq import least_busy
from qiskit.tools.jupyter import *
from qiskit.tools.monitor import job_monitor
from qiskit.visualization import *
from qiskit.ignis.mitigation.measurement import complete_meas_cal, CompleteMeasFitter


provider = IBMQ.load_account() # load your IBM Quantum Experience account
# If you are a member of the IBM Q Network, fill your hub, group, and project information to
# get access to your premium devices.
# provider = IBMQ.get_provider(hub='', group='', project='')

from may4_challenge.ex2 import get_counts, show_final_answer

num_qubits = 5
meas_calibs, state_labels = complete_meas_cal(range(num_qubits), circlabel='mcal')
```

Next, run these circuits on a real device! You can choose your favorite device, but we recommend choosing the least busy one to decrease your wait time in the queue. Upon executing the following cell you will be presented with a widget that displays all the information about the least busy backend that was selected.  Clicking on the "Error Map" tab will reveal the latest noise information for the device.  Important for this challenge is the "readout" (measurement) error located on the left (and possibly right) side of the figure.  It is common to see readout errors of a few percent on each qubit.  These are the errors we are mitigating in this exercise.


```python
# find the least busy device that has at least 5 qubits
backend = least_busy(provider.backends(filters=lambda x: x.configuration().n_qubits >= num_qubits and 
                                   not x.configuration().simulator and x.status().operational==True))
backend
```


    VBox(children=(HTML(value="<h1 style='color:#ffffff;background-color:#000000;padding-top: 1%;padding-bottom: 1…





    <IBMQBackend('ibmq_vigo') from IBMQ(hub='ibm-q', group='open', project='main')>



Run the next cell to implement all of the above steps. In order to average out fluctuations as much as possible, we recommend choosing the highest number of shots, i.e., `shots=8192` as shown below.

The call to `transpile` maps the measurement calibration circuits to the topology of the backend being used. `backend.run()` sends the circuits to the IBM Quantum device returning a `job` instance, whereas `%qiskit_job_watcher` keeps track of where your submitted job is in the pipeline. 


```python
# run experiments on a real device
shots = 8192
experiments = transpile(meas_calibs, backend=backend, optimization_level=3)
job = backend.run(assemble(experiments, shots=shots))
print(job.job_id())
%qiskit_job_watcher
```

    5eb04ddf9f981a00121e28b5



    Accordion(children=(VBox(layout=Layout(max_width='710px', min_width='710px')),), layout=Layout(max_height='500…



    <IPython.core.display.Javascript object>


Note that you might be in the queue for quite a while. You can expand the 'IBMQ Jobs' window that just appeared in the top left corner to monitor your submitted jobs. Make sure to keep your job ID in case you ran other jobs in the meantime. You can then easily access the results once your job is finished by running

```python
job = backend.retrieve_job('YOUR_JOB_ID')
```
    
Once you have the results of your job, you can create the calibration matrix and calibration plot using the following code. However, as the counts are given in a dictionary instead of a matrix, it is more convenient to use the measurement filter object that you can directly apply to the noisy counts to receive a dictionary with the mitigated counts.


```python
job = backend.retrieve_job('5eb04ddf9f981a00121e28b5')
job
```




    namespace(_backend=<IBMQBackend('ibmq_vigo') from IBMQ(hub='ibm-q', group='open', project='main')>,
              _job_id='5eb04ddf9f981a00121e28b5',
              _creation_date=datetime.datetime(2020, 5, 4, 17, 16, 15, 763000, tzinfo=datetime.timezone.utc),
              _api_status=<ApiJobStatus.COMPLETED: 'COMPLETED'>,
              _qobj=None,
              allow_object_storage=True,
              kind=<ApiJobKind.QOBJECT_STORAGE: 'q-object-external-storage'>,
              _time_per_step={'CREATING': '2020-05-04T17:16:15.763Z',
                              'CREATED': '2020-05-04T17:16:16.820Z',
                              'VALIDATING': '2020-05-04T17:16:16.867Z',
                              'VALIDATED': '2020-05-04T17:16:17.666Z',
                              'QUEUED': '2020-05-04T17:16:17.844Z',
                              'RUNNING': '2020-05-04T17:20:47.173Z',
                              'COMPLETED': '2020-05-04T17:23:25.600Z'},
              _error=None,
              _run_mode='fairshare',
              _name=None,
              _result=None,
              _tags=[],
              _backend_info=namespace(name='ibmq_vigo',
                                      id='5d2c7d5ff148e40073c3abe6'),
              cost=314.912,
              summary_data={'size': {'input': 27394, 'output': 17275},
                            'summary': {'partial_validation': False,
                             'gates_executed': 80,
                             'max_qubits_used': 5,
                             'qobj_config': {'shots': 8192,
                              'max_credits': None,
                              'memory_slots': 5,
                              'type': 'QASM',
                              'memory': False,
                              'n_qubits': 5},
                             'num_circuits': 32},
                            'success': True,
                            'resultTime': 157.9890568330884},
              end_date='2020-05-04T17:23:25.410Z',
              share_level='none',
              user_id='5eafec7dd444550019bd097b',
              hub_info={'hub': {'name': 'ibm-q'},
                        'group': {'name': 'open'},
                        'project': {'name': 'main'}},
              object_storage_info={},
              _api=<qiskit.providers.ibmq.api.clients.account.AccountClient at 0x7f81c3b82d10>,
              _use_object_storage=True,
              _queue_info=None,
              _status=<JobStatus.DONE: 'job has successfully run'>,
              _cancelled=False,
              _job_error_msg=None)




```python
# get measurement filter
cal_results = job.result()
meas_fitter = CompleteMeasFitter(cal_results, state_labels, circlabel='mcal')
meas_filter = meas_fitter.filter
print(meas_fitter.cal_matrix.shape)
meas_fitter.plot_calibration()
```

    (32, 32)



![png](images/output_8_1.png)


In the calibration plot you can see the correct outcomes on the diagonal, while all incorrect outcomes are off-diagonal. Most of the latter are due to T1 errors depolarizing the states from $|1\rangle$ to $|0\rangle$ during the measurement, which causes the matrix to be asymmetric.

Below, we provide you with an array of noisy counts for four different circuits. Note that as measurement error mitigation is a device-specific error correction, the array you receive depends on the backend that you have used before to create the measurement filter.

**Apply the measurement filter in order to get the mitigated data. Given this mitigated data, choose which error-free outcome would be most likely.**

As there are other types of errors for which we cannot correct with this method, you will not get completely noise-free results, but you should be able to guess the correct results from the trend of the mitigated results.

## i) Consider the first set of noisy counts:


```python
# get noisy counts
noisy_counts = get_counts(backend)
plot_histogram(noisy_counts[0])
```




![png](images/output_11_0.png)




```python
# apply measurement error mitigation and plot the mitigated counts
mitigated_counts_0 = meas_filter.apply(noisy_counts[0])
plot_histogram([mitigated_counts_0, noisy_counts[0]])
```




![png](images/output_12_0.png)



## Which of the following histograms most likely resembles the *error-free* counts of the same circuit?
a) <img src="images/hist_1a.png" width="500"> 
b) <img src="images/hist_1b.png" width="500"> 
c) <img src="images/hist_1c.png" width="500"> 
d) <img src="images/hist_1d.png" width="500">


```python
# uncomment whatever answer you think is correct
#answer1 = 'a'
#answer1 = 'b'
answer1 = 'c'
#answer1 = 'd'
```

## ii) Consider the second set of noisy counts:


```python
# plot noisy counts
plot_histogram(noisy_counts[1])
```




![png](images/output_16_0.png)




```python
# apply measurement error mitigation
mitigated_counts_1 = meas_filter.apply(noisy_counts[1])
# insert your code here to do measurement error mitigation on noisy_counts[1]
plot_histogram([mitigated_counts_1, noisy_counts[1]])
```




![png](images/output_17_0.png)



## Which of the following histograms most likely resembles the *error-free* counts of the same circuit?
a) <img src="images/hist_2a.png" width="500"> 
b) <img src="images/hist_2b.png" width="500"> 
c) <img src="images/hist_2c.png" width="500"> 
d) <img src="images/hist_2d.png" width="500">


```python
# uncomment whatever answer you think is correct
#answer2 = 'a'
#answer2 = 'b'
#answer2 = 'c'
answer2 = 'd'
```

## iii) Next, consider the third set of noisy counts:


```python
# plot noisy counts
plot_histogram(noisy_counts[2])
```




![png](images/output_21_0.png)




```python
# apply measurement error mitigation
mitigated_counts_2 = meas_filter.apply(noisy_counts[2])
# insert your code here to do measurement error mitigation on noisy_counts[2]
plot_histogram([mitigated_counts_2, noisy_counts[2]])
```




![png](images/output_22_0.png)



## Which of the following histograms most likely resembles the *error-free* counts of the same circuit?
a) <img src="images/hist_3a.png" width="500"> 
b) <img src="images/hist_3b.png" width="500"> 
c) <img src="images/hist_3c.png" width="500"> 
d) <img src="images/hist_3d.png" width="500">


```python
# uncomment whatever answer you think is correct
#answer3 = 'a'
answer3 = 'b'
#answer3 = 'c'
#answer3 = 'd'
```

## iv) Finally, consider the fourth set of noisy counts:


```python
# plot noisy counts
plot_histogram(noisy_counts[3])
```




![png](images/output_26_0.png)




```python
# apply measurement error mitigation
mitigated_counts_3 = meas_filter.apply(noisy_counts[3])
# insert your code here to do measurement error mitigation on noisy_counts[3]
plot_histogram([mitigated_counts_3, noisy_counts[3]])
```




![png](images/output_27_0.png)



## Which of the following histograms most likely resembles the *error-free* counts of the same circuit?
a) <img src="images/hist_4a.png" width="500"> 
b) <img src="images/hist_4b.png" width="500"> 
c) <img src="images/hist_4c.png" width="500"> 
d) <img src="images/hist_4d.png" width="500">


```python
# uncomment whatever answer you think is correct
#answer4 = 'a'
answer4 = 'b'
#answer4 = 'c'
#answer4 = 'd'
```

The answer string of this exercise is just the string of all four answers. Copy and paste the output of the next line on the IBM Quantum Challenge page to complete the exercise and track your progress.


```python
# answer string
show_final_answer(answer1, answer2, answer3, answer4)
```



<p style="border: 2px solid black; padding: 2rem;">
    Congratulations 🎉! Submit the following text
    <samp
        style="font-family: monospace; background-color: #eee;"
    >cdbb</samp>
    on the
    <a href="https://quantum-computing.ibm.com/challenges/4anniversary/?exercise=2&amp;answer=cdbb" target="_blank">
        IBM Quantum Challenge page
    </a>
    to see if you are correct.
</p>



Now that you are done, move on to the next exercise!
