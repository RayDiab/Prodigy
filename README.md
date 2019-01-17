# Project Prodigy

# Table of Contents 

1. [ Project Description. ](#desc)
2. [ Usage Instructions ](#usage)
3. [ Translating ](#trans)
4. [ Neural Network](#network)
5. [ Other ](#other)

<a name="desc"></a>
## 1. Project Description 

Our aim is to train a machine learning algorithm for music composition. For this, we used mlpack, a C++ library, to build a recurrent neural network with a LSTM layer. Since training is more optimal when using number, we wrote an algorithm (with the help of external libraries) (*is this true tranlating people?*) that translates and retranslates MIDI files into csv files containing integers. After the LSTM was build and translating processes was completed, we trained our network using Bach MIDI files located in the folder, *insert here*. A sample composition is uploaded in *insert here* and the some training weights is saved in the folder *insert here* if the user wants readily to compose music. 

For the course of the project we thereby divided ourselves into two teams, the "Translators" and the "Builders". For more information on the respective parts, see the translating  and neural network sections. 

*section about GUI if applicable* 

If you do not have any particular software to play MIDI files, you can either translate them into mp3 using the following [website](https://www.onlineconverter.com/midi-to-mp3) - it looks like some websites do not convert all types of MIDI files properly, but this one has been working well so far - or use a specialized softwares like the [TiMidity++](http://timidity.sourceforge.net/) software synthesiser, which can also create wave audio files from MIDI songs.

<a name="usage"></a>
## 2. Usage Instructions

### Installing the dependencies
After cloning from the project, you first need to install [mlpack](https://www.mlpack.org) and its dependencies.
To do so, simply run the following command:
```
$> bash install_mlpack.bash 
```

### Build and compile
To build the project, run the following commands:
```
$> mkdir build
$> cd build
$> cmake ../
```

Then, to compile the project, enter the `build` folder and type make.
```
$> cd build
$> make
```

### Running the project
Now, the project will have two executables `train` and `compose` which you can execute with the commands 
```
$> ./train
$> ./compose
```
You can either train a model from scratch or continue training on a saved model saved `/utils/model.xml` by executing `train`.
You will need to have a training file `/utils/training.csv` that is a vector of integers corresponding to translated music.

Then, you can generate compositions from the saved model by executing `compose`.

<a name="trans"></a>
## 3. Translating 

### Decision-Making on format
The translating part was about finding a way to get from a audio type input and return an output for the Neural Network, as well as finding a way to get from the outputted format of Neural Network back to an audio type file. We decided to choose midi files as those files are easily found on internet which means more available training data. While midi file was chosen to represent audio files, we decided to have the scores stored as csv files, since csv datas are easily accessible and is also what the Neural Network team needed as input. 

### Ways to go from midi to csv and vice versa
Looking up on internet, we made the decision to use a given library found on github written in C, which was fortunately 
close to c++, and thus automatically encapsulated. The functions from this library are inside midicsv-csvmidi and it suffices to "make" to get the following executables: midicsv/csvmidi which are receiving two arguments the input filename and output filename. These two codes will translate the midi files into csv files of 5 colums detailing everything that is necessary to build back a csv file, that is, in order: 

- The number of the track.
- The number of MIDI ticks associated to the notes (time unit).
- The number associated to the instrument played - here, we only use piano compositions.
- The action applied on the note (Note_On_c = activate note and Note_Off_c = stop playing the note).
- The velocity of the note.


For more information on these csv formats, you can check the following [page](http://www.fourmilab.ch/webtools/midicsv/)

### Format of the input for Neural Networks
In order to feed the Neural Network, we had to simplify the result given by midicsv. We decided to use sequences of integers as the input of our LSTM, as these are easier to handle by the Neural network than a sequence of notes and chords. With this in mind, we tried to simplify the csv associated to our MIDI file to translate by iterating over it and decomposing the partition into a list of consecutive notes and chords (chord = several notes played together). The idea was to look at all the notes that were ON at each MIDI tick and remove the notes that were put OFF at this MIDI tick. The notes that were not eliminated through this process were considered as part of one chord. 

Then, in order to get our sequence of integers, we had to use a specific "translation map" that maps every note or chord to a unique integer. Note that this map cannot be made in advance and has to be created and updated through the translation process are there are virtually an infinite possible number of chords. 

We then use this translation map to form our sequence of integers, which will be written again in a csv format, ready to be sent to the LSTM. 

This part corresponds to the translate.cpp code and and some parts of the translate.hpp code.

Also, as we wanted to translate a large amount of data, we also programmed merge.cpp that merges the midi files together and translates the associated midi file into our sequence of integers.

### The process of backward translation
Now, we had to find a way to translate back the output of the lstm (a sequence of integers) into a midi file that we could listen to. To do so, we had to use the same translation map that was build during the translation process to associate back each integer to a single note or chord. Then, we had to write back the csv file containing the midi ticks, the actions (Note_on and Note_off) executed on each note and the note velocities. 
This can be done with some precision, especially for the sequence of notes, but it is important to understand that a lot of information about the initial music has been lost through our translation process: we no longer know the exact time each action was made nor the velocity of each note. In this perspective, we now had to do the following approximations:

- We consider the time interval between two consecutive midi ticks to be equal. 
- We consider the velocity of the notes to be equal.
- Also, some notes may have been lost or shifted in our translation process.

Therefore, using the same translation map as before and taking into account the above approximations, we can then translate back the output of the lstm into a csv understandable by the csvmidi code mentioned above, which can transform it back into midi files. This part corresponds to the transelatebackv.cpp code and some part of the translate.hpp code.

### Accuracy evaluation of the backward translation algorithm

Once we had both algorithms, we wanted to assess more formally the accuracy of the backward translation algorithm, that is to say how the approximation we chose to make would change the file. The algorithm takes as input a csv file of a music translated by “midicsv” (not transformed) and a csv file translated both ways by “translate.cpp” and “translateback.cpp”. It outputs the average difference between two consecutive ticks' time intervals between the original file and the translated one and the average difference in velocity between the original file and the translated one. It was a way to quantify the lost but also to compare composers like Mozart and Bach, in order to see which one fitted our project the best. The algorithm also outputs the number of identical similar notes between the two files. As expected, some notes are lost, which creates a shift between the two files that complicates their analysis but doesn't really change the sound. 
 


<a name="network"></a>
## 4. Neural Network 
### Music composition with recurrent neural networks (RNN)
We chose to implement an artificial neural network for its sheer prediction power after enough training. Because of the sequential nature of music, the dependency of the value of the notes at a certain time step to the music that precedes it, a recurrent neural network rather than a feedforward neural network is the necessary choice.

More specifically, we implemented a Long short-term memory network (LSTM). This [article](http://colah.github.io/posts/2015-08-Understanding-LSTMs/) gives a through explanation of LSTM networks, but the main idea is that LSTM units in our RNN will be able to recognize and learn long-time patterns, which is what we need for music composition.

We decided to use an external C++ library for machine learning, [mlpack](https://www.mlpack.org) to build and train our LSTM network.

### Structure of our model
We defined two hidden LSTM layers with 512 memory units, and a dropout layer of probability 0,3 which helps avoid overfitting. The output layer uses the softmax activation function to output the log of the probability prediction for each notes present in our model.
The problem can be defined as a single integer classification problem with each note being a possible class, therefore training is minimizing the [negative log-likelihood](https://ljvmiranda921.github.io/notebook/2017/08/13/softmax-and-the-negative-log-likelihood/), which maximizes the likelihood that the output of the model produces the data actually observed. We also use the ADAM optimization algorithm for speed.

### Format of our data
In general, LSTM networks expect input data with different features, time step and points. Specifically, in the mlpack library, the LSTM layer takes in an armadillo cube where each row corresponds to a "feature", in our case we only have one feature that is the note, each column corresponds to a "time step" which is the point in time within our sequence of music, and each slice (the third dimension of our tensor) corresponds to a point, or the specific sequence of notes at the time step considered.

The training labels or what the model is expected to output is defined as the single note that follows each sequence of notes considered at each time step. Here, it is an armadillo cube again with the same dimensions as the training data, where each column or time step stores the note corresponding to the time step.
We use the entirety the music data for training, we do not have a validation set as it does not make sense in the context where we want the model to learn the probabilities of notes given a musical sequence.

The output of our model is a cube with one row, as many columns as the total number of different notes, and as many slices as the length of the sequence considered at each time step. To extract the notes given a probability vector, we simply look for the index with the maximum probability and choose that as the note predicted by our model. 

We measure accuracy after each cycle of training by calculating the percentage of notes from the prediction that coincide with the note from the training set at the respective time step. Note that this accuracy is only one indicator to keep track of training, and we expect the percentage to stay low as the model should still have some creativity. This accuracy mostly ensures no overfitting happens, as we do not want the model to reproduce exactly the given training music. 

### Generating music after training
Once we have trained a model, we can save the model with the trained weights for later use. We have chosen to have the model be saved automatically after each 20 cycles of training.

To have the network generate music, we then load the previously trained model. Essentially, the trained network is a prediction model, so it needs a starting point for composition. We use a randomly generated short sequence of notes as the seed sequence which we feed into the prediction method of the model. From then on, we feed the new sequence predicted by the model to obtain the next predicted sequence, and we continue this step until we get a music sequence of our desired length.

<a name="extra"></a>
## 5. What now?

Here are some extension ideas that have not been explored that could possibly improve our implementation:
* Adding a temperature layer before the logsoftmax layer, to control the randomness of predictions by the LSTM. 

Temperature will rescale the logits before putting them through the softmax function, where high temperature will give similar probabilities to each notes and low temperature will give higher probability to the most expected note. We can therefore think of temperature as the parameter of the LSTM's creativity, where a model with low temperature will be more "creative".
* Single-note training

In our project, each combination of notes from the training set were encoded by a unique integer, which limits the combinations of notes the model can produce to ones it has already seen during training. Another way to encode music would be to give each note a unique character, then encode combinations of notes as combinations of these characters. This encoding is more complex but would be a more realistic modelization of true music composition.

Furthermore, we can add complexity by finding a better way to encode rhytm. Currently, the scope of our training music is limited to music containing notes with lower velocities. This is due to the way we translation of the MIDI files. In particularly the tick like representation of music and the duration of time that note is help adds complexity to the translating dictionary. 
*translating hu,man verify the above/make more complete please* 

* Improve measure used to keep track of training

The accuracy measure we implemented is a very naive measure where we simply calculate the percentage of notes the model outputs given the training set, to the actual notes from the training set. This measure is misleading and not well-suited in the scope of this project, as a model we would consider very good at producing music does not, and should not even have a high accuracy during training. This leads to a fundamental question of how to classify what music is considered as "good". Answering such a question might require developing a criteria based on musical theory. 





