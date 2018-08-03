# Age and Gender Detection using OpenVINO™
### Lab Overview
We have done Face Detection in our previous module. Now, we identify Age and Gender for the identified faces.    
We  build upon our Face Detection code and add Age, Gender identification code in this module.
### Class diagram for AgeGender detection
![](images/AgeGender_class.png)

### We replace Age and Gender Detection TODOs with the following:
-	Include CPU as plugin device for parallel inferencing.
-	Load pre-trained data model for Age and Gender detection.
-	Once Face Detection result is available, submit inference request for Age and Gender Detection
-	Mark the identified faces inside rectangle and put text on it for Age and Gender.
-	Observe Age and Gender Detection in addition to face.

![](images/AgeGender_flowchart.png)

### AgeGenderDetection structure
We need to define a class which includes member variables and member functions that will be used for AgeGender detection using OpenVINO™.
- Replace #TODO: AgeGenderDetection structure
- Paste the following code

```
struct AgeGenderDetection {
	std::string input;
	std::string outputAge;
	std::string outputGender;
	int enquedFaces = 0;


	ExecutableNetwork net;
	InferenceEngine::InferencePlugin * plugin;
	InferRequest::Ptr request;
	std::string & commandLineFlag = FLAGS_Age_Gender_Model;
	std::string topoName = "Age Gender";
	const int maxBatch = FLAGS_n_ag;



	void submitRequest()  {
		if (!enquedFaces) return;
		request->StartAsync();
		enquedFaces = 0;
	}

	void wait() {
		if (!request) return;
		request->Wait(IInferRequest::WaitMode::RESULT_READY);
	}

	void matU8ToBlob(const cv::Mat& orig_image, Blob::Ptr& blob, float scaleFactor = 1.0, int batchIndex = 0) {
	SizeVector blobSize = blob.get()->dims();
	const size_t width = blobSize[0];
	const size_t height = blobSize[1];
	const size_t channels = blobSize[2];

	float* blob_data = blob->buffer().as<float*>();

	cv::Mat resized_image(orig_image);
	if (width != orig_image.size().width || height != orig_image.size().height) {
		cv::resize(orig_image, resized_image, cv::Size(width, height));
	}

	int batchOffset = batchIndex * width * height * channels;

	for (size_t c = 0; c < channels; c++) {
		for (size_t h = 0; h < height; h++) {
			for (size_t w = 0; w < width; w++) {
				blob_data[batchOffset + c * width * height + h * width + w] =
					resized_image.at<cv::Vec3b>(h, w)[c] * scaleFactor;
			}
		}
	}
}


	void enqueue(const cv::Mat &face) {

		if (!request) {
			request = net.CreateInferRequestPtr();
		}

		auto  inputBlob = request->GetBlob(input);
		matU8ToBlob(face, inputBlob, 1.0f, enquedFaces);
		enquedFaces++;
	}

	struct Result { float age; float maleProb; };
	Result operator[] (int idx) const {
		auto  genderBlob = request->GetBlob(outputGender);
		auto  ageBlob = request->GetBlob(outputAge);

		return{ ageBlob->buffer().as<float*>()[idx] * 100,
			genderBlob->buffer().as<float*>()[idx * 2 + 1] };
	}

	void into(InferenceEngine::InferencePlugin & plg)  {

			net = plg.LoadNetwork(this->read(), {});
			plugin = &plg;

	}

	CNNNetwork read()  {

		InferenceEngine::CNNNetReader netReader;
		/** Read network model **/
		netReader.ReadNetwork(FLAGS_Age_Gender_Model);

		//	/** Set batch size to 16
		netReader.getNetwork().setBatchSize(16);

		/** Extract model name and load it's weights **/
		std::string binFileName = fileNameNoExt(FLAGS_Age_Gender_Model) + ".bin";
		netReader.ReadWeights(binFileName);

		/** Age Gender network should have one input two outputs **/
		InferenceEngine::InputsDataMap inputInfo(netReader.getNetwork().getInputsInfo());

		auto& inputInfoFirst = inputInfo.begin()->second;
		inputInfoFirst->setPrecision(Precision::FP32);
		inputInfoFirst->getInputData()->setLayout(Layout::NCHW);
		input = inputInfo.begin()->first;

		// ---------------------------Check outputs ------------------------------------------------------
		InferenceEngine::OutputsDataMap outputInfo(netReader.getNetwork().getOutputsInfo());

		auto it = outputInfo.begin();
		auto ageOutput = (it++)->second;
		auto genderOutput = (it++)->second;

		outputAge = ageOutput->name;
		outputGender = genderOutput->name;
		return netReader.getNetwork();
	}
};

```
### Include CPU as Plugin Device
We need to include CPU as plugin device for inferencing Age and Gender
- Replace #TODO: Age and Gender detection 2
- Paste the following lines
```
plugin = PluginDispatcher({ "../../../lib/intel64", "" }).getPluginByDevice("CPU");
pluginsForDevices["CPU"] = plugin;
  ```

### Load Pre-trained Optimized Model for Age and Gender Inferencing
We need CPU as plugin device for inferencing Age and Gender and load pre-retained model for Age and Gender Detection on CPU
- Replace #TODO: Age and Gender Detection 3
- Paste the following lines
```
FLAGS_Age_Gender_Model = "/opt/intel/computer_vision_sdk_2018.1.265/deployment_tools/intel_models/age-gender-recognition-retail-0013/FP32/age-gender-recognition-retail-0013.xml";
AgeGenderDetection AgeGender;
Load(AgeGender).into(pluginsForDevices["CPU"]);
 ```

### Submit Inference Request
- Replace #TODO: Age and Gender Detection 4
- Paste the following lines
```
    //Submit Inference Request for age and gender detection and wait for result
	AgeGender.submitRequest();
	AgeGender.wait();
 ```

### Use Identified Face for Age and Gender Detection
Clip the identified Faces and send inference request for identifying Age and Gender
- Replace #TODO: Age and Gender Detection 5
- Paste the following lines
```
    //Clipped the identified face and send Inference Request for age and gender detection
	for (auto face : FaceDetection.results) {
		auto clippedRect = face.location & cv::Rect(0, 0, 640, 480);
		auto face1 = frame(clippedRect);
		AgeGender.enqueue(face1);
	}
	// Got the Face, Age and Gender detection result, now customize and print them on window
	std::ostringstream out;
	index = 0;
	curFaceCount = 0;
    malecount=0;
	femalecount=0;
 ```

### Customize the Result for Display
Now we got result for Face, Age and Gender detection. We can customize the output and display this on the screen
- Replace #TODO: Age and Gender Detection 6
- Paste the following lines
```
            out.str("");
			curFaceCount++;

			//Draw rectangle bounding identified face and print Age and Gender
			out << (AgeGender[index].maleProb > 0.5 ? "M" : "F");
			if(AgeGender[index].maleProb > 0.5)
				malecount++;
			else
				femalecount++;
			out << "," << static_cast<int>(AgeGender[index].age);

			cv::putText(frame,
				out.str(),
				cv::Point2f(result.location.x, result.location.y - 15),
				cv::FONT_HERSHEY_COMPLEX_SMALL,
				0.8,
				cv::Scalar(0, 0, 255));

			index++;

 ```

### The Final Solution
Keep the TODOs as it is. We will re-use this program during Cloud Integration.     
For complete solution click on following link [Age_gender.cpp](./script/agegenderdetection.md)
### Build the Solution and Observe the Output
- Go to ***~/Desktop/Retail/OpenVINO/samples/build***  directory
- Do  make by following commands   
- Make sure environment variables set when you are doing in fresh terminal.      

```
make
 ```
- Executable will be generated at ***~/Desktop/Retail/OpenVINO/samples/build/intel64/Release*** directory.
- Run the application by using below command. Make sure camera is connected to the device.

```
./interactive_face_detection_sample
 ```
![](images/openvinoface.jpg)
### Lesson Learnt
In addition to Face, Age and Gender Detection using Intel® OpenVINO™ toolkit.