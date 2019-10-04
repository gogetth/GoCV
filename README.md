# GoCV

The GoCV package supports the latest releases of Go and OpenCV v3.4.2 on Linux, macOS, and Windows. We intend to make the Go language a “first-class” client compatible with the latest developments in the OpenCV ecosystem.

---
## Install GoCV => Linux/macOS/Windows
https://gocv.io/getting-started/

---
## Test Example GoCV
```
git clone https://github.com/gogetth/GoCV.git
cd GoCV
```

* Hello Video

        go run hellovideo.go

* Face Detect

        go run facedetect.go 0 data/haarcascade_frontalface_default.xml

---
## Ref. [GoCV](https://gocv.io/)

# Real-time object detection with Go, Tensorflow, and OpenCV

## About

This is a small demo app using Go, Tensorflow, and OpenCV to detect objects objects in real time using the Google provided Tensorflow Object Dection API project models.  This project is primarily an example of gluing all of the components together into a functional demo that should be relatively cross platform, though there are likely numerous optimizations that could be made.

Additionally, one of my goals was to explore the viability of Go as an option for a local high performance app with a UI (instaed of C++ or Java), and I'm happy with the initial results.

## Components

Webcam capture and image pre-processing is done with [GoCV](https://github.com/hybridgroup/gocv).

Each captured frame is run in realtime through a model from the [Tensorflow Object Dection API](https://github.com/tensorflow/models/tree/master/research/object_detection) project, which outputs object bounding boxes and confidence ratings.

The prediction code is laregely adapted from https://github.com/ActiveState/gococo.

Most of my testing has been with the `ssd_mobilenet_v1_coco` model as it runs well without a GPU. Additional models are available in the [model detection zoo]( https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md).

This project uses [Pixel](https://github.com/faiface/pixel) to display the image and render bounding boxes and labels using OpenGL.  This isn't strictly neccessary, but I find the additional tools provided by Pixel useful (somewhat akin to the openFrameworks toolbox for c++).  It could easily be swapped out, like for a [mjpeg http stream](https://github.com/hybridgroup/gocv/tree/master/cmd/mjpeg-streamer) and a web ui (for example).

## Performance

On a 2015 Macbook Pro using `ssd_mobilenet_v1_coco` and CPU-only inference I can get ~15-20 fps.  The non-[mobilenet](https://research.googleblog.com/2017/06/mobilenets-open-source-models-for.html) models are of course much slower without a GPU.  Compiling tensorflow with AVX support gave a boost of 2-3 fps.  Additional speed can be gained [tuning the model score_threshold and re-exporting](https://github.com/tensorflow/models/issues/1609#issuecomment-309502384).  Also, you should be able to link the GPU Tensorflow library without needing to change anything in this project.

## Installation

I've not vendored dependencies as they rely on system level C libraries, which makes things a bit funky.  It's doable, but not worth it for a demo.

1. Git clone this repo
2. Install gocv and OpenCV following these instruction: https://github.com/hybridgroup/gocv#how-to-install
3. Install Tensorflow and the go bindings: https://www.tensorflow.org/install/install_go
   - If you want to build from source follow this doc https://github.com/tensorflow/tensorflow/blob/master/tensorflow/go/README.md
     - I used this command to build with extra CPU features : `bazel build -c opt --copt=-mavx --copt=-mavx2 --copt=-mfma --copt=-msse4.2 -k //tensorflow:libtensorflow.so`
4. Install Pixel https://github.com/faiface/pixel#pixel----
4. Download and extract a model from the [model zoo](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md), [`ssd_mobilenet_v1_coco`](http://download.tensorflow.org/models/object_detection/ssd_mobilenet_v1_coco_11_06_2017.tar.gz) is a good starting point.
5. `go run src/main.go -device 0 -dir <path_to_model_dir>`


You should be able to execute something like this on osx:
```sh
# this repo
git clone https://github.com/gogetth/GoCV
cd GoCV
# gocv
brew install opencv
go get -u -d gocv.io/x/gocv
source $GOPATH/src/gocv.io/x/gocv/env.sh
# tensorflow
TF_TYPE="cpu" # Change to "gpu" for GPU support
 TARGET_DIRECTORY='/usr/local'
 curl -L \
   "https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-${TF_TYPE}-$(go env GOOS)-x86_64-1.3.0.tar.gz" |
 sudo tar -C $TARGET_DIRECTORY -xz
go get github.com/tensorflow/tensorflow/tensorflow/go
# pixel
go get github.com/faiface/pixel
# get the ssd mobilenet model
mkdir -p models
curl -L "http://download.tensorflow.org/models/object_detection/ssd_mobilenet_v1_coco_11_06_2017.tar.gz" | tar -C models -xz
# run it
go run src/main.go -device 0 -dir models/ssd_mobilenet_v1_coco_11_06_2017
```

## Error
src/main.go:122:20: cannot use frame (type gocv.Mat) as type *gocv.Mat in argument to cam.Read
src/main.go:127:14: cannot use frame (type gocv.Mat) as type *gocv.Mat in argument to gocv.Resize

- (Old Version)
```go
func capture(deviceID int, frames chan []byte) {
	cam, err := gocv.VideoCaptureDevice(deviceID)
	if err != nil {
		log.Fatal("failed reading cam")
	}
	defer cam.Close()

	frame := gocv.NewMat()
	defer frame.Close()

	for {
		if ok := cam.Read(frame); !ok {
			log.Fatal("failed reading cam")
		}

		// Resize the Mat in place.
		gocv.Resize(frame, frame, image.Point{X: W, Y: H}, 0.0, 0.0, gocv.InterpolationNearestNeighbor)

		// Encode Mat as a bmp (uncompressed)
		buf, err := gocv.IMEncode(".bmp", frame)
		if err != nil {
			log.Fatalf("Error encoding frame: %v", err)
		}

		// Push the frame to the channel
		frames <- buf
	}
}
```

- (New version) [Edit variable `frame` line `122` and `127` on file `main.go` in `src`]

```go
func capture(deviceID int, frames chan []byte) {
	cam, err := gocv.VideoCaptureDevice(deviceID)
	if err != nil {
		log.Fatal("failed reading cam")
	}
	defer cam.Close()

	frame := gocv.NewMat()
	defer frame.Close()

	for {
		if ok := cam.Read(&frame); !ok {
			log.Fatal("failed reading cam")
		}

		// Resize the Mat in place.
		gocv.Resize(frame, &frame, image.Point{X: W, Y: H}, 0.0, 0.0, gocv.InterpolationNearestNeighbor)

		// Encode Mat as a bmp (uncompressed)
		buf, err := gocv.IMEncode(".bmp", frame)
		if err != nil {
			log.Fatalf("Error encoding frame: %v", err)
		}

		// Push the frame to the channel
		frames <- buf
	}
}
```