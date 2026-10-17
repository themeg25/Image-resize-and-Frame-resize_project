S3 Image Resizer (AWS Lambda)

An AWS Lambda function that automatically creates high-quality, fixed-size JPEG versions of images uploaded to an S3 bucket. Originals are never modified or deleted.

For every image uploaded to Test-Original-Images/, the function writes two resized copies:

Output folder	Display size (W x H)	Saved file size (at OUTPUT_SCALE = 3)
150*150/	150 x 150	450 x 450
200*400/	400 x 200	1200 x 600

Note: the folder name 200*400/ is kept for backward compatibility, but the image inside is 400 wide x 200 high.

Features
S3 event trigger: processes images automatically on upload.
Backfill mode: process every image already in the input folder with one manual invocation.
Sharp output: images are saved at 3x the display size so they stay crisp when zoomed (configurable).
Light sharpening: a subtle unsharp mask applied after resizing, with no halos or color shifts.
Color preserved: no contrast or color adjustments; the original ICC profile is carried over.
EXIF orientation fixed: phone photos are rotated correctly.
Transparency handled: PNG/GIF/WebP transparency is flattened onto a white background.
Animated images: only the first frame is used.
Safe re-processing: output objects are tagged with metadata, and already-processed inputs are skipped.
Huge-image guard: sources larger than 8000 x 8000 px are downscaled before processing.
How it works
Upload to  Test-Original-Images/photo.png
        |
        v
  S3 event  ->  Lambda
        |
        |-- validate image (Pillow verify)
        |-- fix EXIF rotation, convert to RGB
        |-- resize to each target size
        |-- sharpen lightly
        |-- encode as JPEG (quality 96, 4:4:4)
        v
  150*150/photo.jpg
  200*400/photo.jpg

Sub-folders are preserved. Test-Original-Images/shirts/a.png becomes 150*150/shirts/a.jpg and 200*400/shirts/a.jpg.

Resize behavior

Both outputs use the same "no-cut frame fill" logic: the whole image is expanded to fill the whole frame. There are no borders, no padding and no cropping.

Important: because nothing is cropped, an image whose aspect ratio differs from the target will be stretched to fit. A wide photo in the 150x150 square will look squeezed horizontally, for example.

If you would rather allow a limited stretch and crop the remainder, set max_distortion in the OUTPUTS dictionary:

max_distortion	Behavior
None	Never crop. The full image is stretched to fill the frame.
0.15	Allow up to 15% stretch. If more is needed, crop only the extra part (centered by default).

Crop position is controlled by CROP_CENTER_X and CROP_CENTER_Y (0.5 = center).

Configuration

All settings are constants at the top of the file.

Setting	Default	Description
BUCKET_NAME	stylo-captain-bucket	S3 bucket to read from and write to. Events from other buckets are ignored.
INPUT_PREFIX	Test-Original-Images/	Folder containing the original images.
SQUARE_OUTPUT_PREFIX	150*150/	Output folder for the square version.
RECTANGLE_OUTPUT_PREFIX	200*400/	Output folder for the rectangle version.
OUTPUTS	see code	Map of folder: (display_width, display_height, max_distortion).
OUTPUT_SCALE	3	Pixel multiplier for saved files. Set to 1 for exact 150x150 / 400x200 pixels.
CROP_CENTER_X / CROP_CENTER_Y	0.5	Crop focal point, used only when max_distortion is set.
MAX_SOURCE_WIDTH / MAX_SOURCE_HEIGHT	8000	Larger sources are downscaled first.
JPEG_QUALITY	96	JPEG quality (1-100).
PROCESSED_METADATA_KEY / VALUE	image-resizer-processed / true	Metadata tag used to mark processed objects.
Requirements
Runtime: Python 3.10+ on AWS Lambda
Libraries:
boto3 (already included in the Lambda runtime)
Pillow
numpy
opencv-python-headless

Pillow, numpy and OpenCV are not included in the default Lambda runtime. Package them in a Lambda layer or a container image. Use opencv-python-headless (not opencv-python) so no GUI libraries are required, and build for the same architecture as your function (x86_64 or arm64).

Example layer build (Linux, Python 3.12):

bash
mkdir -p layer/python
pip install pillow numpy opencv-python-headless -t layer/python
cd layer && zip -r ../image-resizer-layer.zip python

Note that Lambda layers have a 250 MB unzipped size limit. If you exceed it, use a container image.

Deployment
Create the Lambda function and upload the code (handler: lambda_function.lambda_handler, assuming the file is named lambda_function.py).
Attach the layer containing Pillow, numpy and OpenCV.
Set resources: memory of 1024-2048 MB and a timeout of 60 seconds or more are recommended for large images.
Add IAM permissions to the function's execution role:
json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["s3:GetObject", "s3:PutObject"],
         "Resource": "arn:aws:s3:::stylo-captain-bucket/*"
       },
       {
         "Effect": "Allow",
         "Action": "s3:ListBucket",
         "Resource": "arn:aws:s3:::stylo-captain-bucket"
       }
     ]
   }

Also attach AWSLambdaBasicExecutionRole for CloudWatch logging. s3:ListBucket is needed for backfill mode.

Add the S3 trigger:
Event type: All object create events (s3:ObjectCreated:*)
Prefix: Test-Original-Images/
Keep the trigger prefix limited to the input folder. The output folders are outside it, so the function never triggers itself.
Usage
Automatic (on upload)

Upload an image to Test-Original-Images/ and the resized versions appear in the output folders within seconds.

bash
aws s3 cp photo.png s3://stylo-captain-bucket/Test-Original-Images/photo.png
Backfill existing images

Invoke the Lambda manually (console Test tab or CLI) with this event:

json
{ "mode": "backfill" }
bash
aws lambda invoke \
  --function-name <your-function-name> \
  --payload '{"mode": "backfill"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

Backfill lists every object under INPUT_PREFIX and processes each one. For very large buckets, consider running it in batches, since a single invocation is limited by the Lambda timeout (maximum 15 minutes).

Response format
json
{
  "statusCode": 200,
  "message": "Image processing completed",
  "results": [
    {
      "input": "Test-Original-Images/photo.png",
      "status": "processed",
      "original_size_kb": 842.17,
      "outputs": [
        {
          "key": "150*150/photo.jpg",
          "width": 450,
          "height": 450,
          "display_width": 150,
          "display_height": 150,
          "size_kb": 38.4,
          "jpeg_quality": 96
        }
      ],
      "message": "Original preserved; resized images created."
    }
  ]
}

Possible status values:

Status	Meaning
processed	Image resized and uploaded successfully.
ignored	Skipped. See reason: unexpected bucket, outside input folder, folder placeholder, already processed, empty file, or invalid/unsupported image.
already_processed	(Backfill) object already carries the processed metadata tag.
failed	(Backfill) an error occurred for this object. See error.
Output details
Format: always JPEG (.jpg), regardless of the source format.
Quality: 96, with 4:4:4 chroma subsampling, optimized and progressive encoding.
Content type: image/jpeg.
Object metadata on each output:
image-resizer-processed: true
source-key: the original object key (truncated to 512 characters)
Same name collisions: logo.png and logo.jpg in the same folder both map to logo.jpg, so the later one overwrites the earlier one.
Troubleshooting
Problem	Likely cause / fix
Unable to import module / No module named cv2	The layer is missing, built for the wrong Python version or architecture, or you installed opencv-python instead of opencv-python-headless.
Timeout or out-of-memory errors	Increase Lambda memory and timeout. Very large source images need more of both.
Images look stretched	Expected with max_distortion = None. Set it to a value such as 0.15 to allow a small crop instead.
Nothing happens on upload	Check that the S3 trigger exists, its prefix matches Test-Original-Images/, and the bucket name matches BUCKET_NAME.
AccessDenied errors	Review the IAM role permissions listed above.
Output is skipped as "Invalid or unsupported image"	The file is corrupt or a format Pillow cannot read (for example, some HEIC files without extra plugins).

Logs are written to CloudWatch under /aws/lambda/<your-function-name>.

Project structure
.
├── lambda_function.py   # Lambda handler and all image-processing logic
└── README.md
