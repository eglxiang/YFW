Experiments on YouTube Face (YTF) dataset. The purpose of this documentary and repository is to make sure that you can replicate the same experiments as I did.

Motivation: We need the sample mean and variation to represent a data distribution while the mean itself is not a robust statistic. However, feature averaging is straightforward and conventional to represent a sequence such as in the recent works of video captioning and activity recognition. We argue that the frame-wise feature mean is unable to characterize the variation existing among frames. For instance, if we want the feature to represent the subject identity of a video, we had better preserve the overall pose diversity. Disregarding factors other than identity and pose, identity will be the only source of variation across videos since pose varies even within a single video. Using a simple metric, the correlation between two video features measures how likely the two videos represent the same person. Following such a variation disentanglement idea, we present a pose-robust face verification algorithm using deeply-learned CNN features. Instead of simply using all the frames, the algorithmis highlighted at the key frame selection using pose distances to K-means centroids, which reduces the number of feature vectors from hundreds to K while still preserving the overall diversity. We also analyze how this video sampling strategy is better than random sampling. On the official 5000 video-pairs of YouTube Face dataset, our algorithm achieves acomparable performance with state-of-the-art that averages over features of all frames.

Background: The idea of Linear Discriminant Analysis (LDA) is minimizing intra-class variance and maximizing inter-class variance, which is general. The adaptation of this idea to similarity learning is interesting, such as Discriminative Component Analysis (DCA), Local Fisher Discriminant Analysis (LFDA) and Logistic Discriminant-based Metric Learning (LDML). In ICCV 2009, Matthieu Guillaumin, Jakob Verbeek and Cordelia Schmid proposed this LDML approach in the paper titled "Is that you? Metric Learning Approaches for Face Identification" which has been cited by hundreds of papers. IDML was tested on the Labeled Face in the Wild (LFW) dataset at that time. Two years later, another dataset called YTF (YouTube Face) was built by using the 5,749 names of subjects included in the LFW dataset to search YouTube for videos of these same individuals. Then, a screening process reduced the original set of videos from the 18, 899 originally downloaded (3,345 individuals) to 3,425 videos of 1,595 subjects.

Video-based face recognition benchmark makes the subsequent solutions closer to a real-world face recognition solution. There should be a fair amount of interests to see the performance of LDML on YTF. As time goes by, Local Binary Feature (LBP) has been mostly replaced by deep learning features in the current experiments on either LFW or YTF. Following the same replacement of feature representation, this repository provides a tutorial to verify the metric learning approach of LDML on YTF.

Please download the full dataset from http://www.cs.tau.ac.il/~wolf/ytfaces/ and cite the original paper at CVPR'11 if you publish the experiments on that dataset. We remove the '2' sequence of 'Lionel_Richel' (see a sample image in this repository) due to really low image quality possibly due to compression artifacts. We remove the '1' sequence of 'Ximena_Bohorquez' (see a sample image in this repository) because half of the detected face profile image is background and image quality is poor. 

Xiang Xiang (eglxiang@gmail.com), January 2016, MIT license.

=========================
Pre-processing.

Input:  YTF dataset - videos of hoslitic scenes containing faces.

Output: sampled dataset - selected frames of faces.

1. Face detection and cropping: cropFace.py and process.py.
The faces in the 'aligned_images_DB' folder of YTF are in the hoslitic scene. However, they have already been centered according to a certain aspect ratio. As a result, the cropping is straightforward.

2. Key face selection by pose quantization: sampleSelect.m and selectImg.py.
Training face selection by K-means clustering the poses which are rotation angles of roll, pitch and yaw, respectively. 
(1) Computing the rotation angles for each face video using existing 3D pose estimation method in OpenCV.
In particular, the 'headpose_DB' of YTF already contain the poses for each frame.
(2) Performs a frame-wise vector quantization which reduces the number of images required to represent the face from tens or hundreds to K (say, K = 9 for a K-means codebook), while preserving the overall diversity.
i) Clustering. Due to the randomness of K-means, you won't get exactly the same result every run.
ii) Selection. Selecting samples using distances from each point to every centroid. Outputing indexes to index.txt and copying images using selectImg.py.

3.  Split training and testing set.
There are 1,595 names in YTF.

(1) Unrestricted protocol: providing labels of subject identity.

i) Splitting YTF by person.
We make sure that the training person set non-overlaps with testing person set. As a result, there are 798 people for training and 797 for testing where each person has at least 1 sequence. Then, we learn a projection matrix from the training data and use it to project testing data into a hopefully more discriminative feature space.
ii) Splitting each person's imagery by sequence. 
Only spliting those with at least 2 sequences (1,003 people). Taking 1 sequence of each person for training; the rest for testing.
The person with only 1 sequence (592 people) will only be used as testing data which will be used to verify the generalisation of the learned metric or simply as a non-of-them class.

(2) Restricted protocol: providing labels of 'same' or 'not-same'.

The goal of this protocol is to determine, for each split, which are the same and which are the non-same pairs, by training on the pairs from the nine remaining splits.
Randomly collecting 5,000 video pairs as listed in http://www.cs.tau.ac.il/~wolf/ytfaces/splits.txt
half of which are pairs of videos of the same person and half of different people.
These pairs were divided into 10 splits.
1~500 is the 1st split: 1~250 is 'same' while 251~500 is 'not-same'.
501-1000 is the 2nd split: 501~750 is 'same' while 751~1000 is 'not-same'.
and so on. 

=========================
Deep Feature Extraction.

Input: Selected faces.

Output: feature representations.

1. Deep face descriptor.
Grab codes from another repository of me - https://github.com/eglxiang/vgg_face.git and then use the main_seq.cpp which process all images in the directory specified in the argument. VGG pre-trained model and the Caffe protocol file can be downloaded from http://www.robots.ox.ac.uk/~vgg/software/vgg_face/src/vgg_face_caffe.tar.gz Features are FC7 so they are 4096-dim. Note that our program will write the feature vector of each face image into a txt file. A pre-complied binary in release mode is also provided (classify_test).

2. Bash processing for YTF.
(1) Read selected images for each sequence over all people in the YTF dataset.
(2) Use comuFea.sh to process a single sequence. You need to explicitly call the binary (either release or debug mode) such as ./bin/Release/classify_test VGG_FACE_deploy.prototxt VGG_FACE.caffemodel $arg3 where arg3 is the argument defined in the bash.

=========================
Pairwise metric learning. 
During the process of metric training, we want to learn the latent basis that CNN features will be project on.
And we hope that once applying the projection to testing sample, it is more discriminative.

Multiple Instance Logistic Discriminant-based Metric Learning (MildML) is an extension of LDML for handling bag-level supervision, using the Multiple Instance Learning framework. Please download the program of MildML from http://lear.inrialpes.fr/people/guillaumin/code/MildML_0.1.tar.gz

function [ L b info ] = ldml_learn( X, Y, k, it, verbose, A0 )
Input: X is a (m x d) data matrix (m data points with d dimensions) and Y is a (m x 1) class labels (1 out of 1595)
where m is , d is 2622 (fc8 is chosen and 2622 corresponds to the number of identities in VGG Face's training set).

================
Note that if you want to save to a directory with another name, you should change it in compuFea2.sh as well as the main.cpp. SImilarly if you want to input images in another directory other than 'selected_faces'.

I am more than happy to know if this program is of use to your work.
Please cite the following paper.

@inproceedings{xiang16pose, 
  author = {Xiang, Xiang and Tran, Trac D},<br>
  Title = {Pose-Selective Max Pooling for Measuring Similarity},<br>
  booktitle = {VAAM/FFER@ICPR 2016},<br>
  Year  = {2016}<br>
}
