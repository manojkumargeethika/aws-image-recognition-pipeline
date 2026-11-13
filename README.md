# CS643 Project 1 – Image Recognition Using AWS

**Student:** Geethika Manojkumar  
**Course:** CS643-851  
**Project:** AWS-Based Car Detection and Text Recognition

## 1. Project Overview

This project implements a distributed image recognition application using Amazon Web Services (AWS). The application processes images stored in an Amazon S3 bucket, identifies images containing cars using Amazon Rekognition, and performs text recognition on qualifying images.

The application uses two EC2 instances that communicate through an Amazon SQS FIFO queue.

The project consists of two Java applications:

- **ObjectDetection.java:** Runs on EC2-A. Retrieves images from the instructor-provided S3 bucket, uses Amazon Rekognition to detect cars, and sends the names of qualifying images to an SQS FIFO queue.
- **TextRecognition.java:** Runs on EC2-B. Receives image names from the SQS queue, retrieves the corresponding images from S3, uses Amazon Rekognition to detect text, and saves the recognized text to an output file.

The application uses a confidence threshold greater than 80% for both car detection and text recognition.

## 2. AWS Architecture

The project uses the following AWS services:

- **Amazon EC2:** Hosts the two Java applications.
- **Amazon S3:** Uses the instructor-provided bucket to store and retrieve the input images.
- **Amazon SQS:** Provides communication between the two EC2 instances through a FIFO queue.
- **Amazon Rekognition:** Performs car detection and text recognition.
- **IAM Role:** Allows the EC2 instances to access the required AWS services without manually configuring access keys.

### Application Workflow

1. EC2-A runs ObjectDetection.java.
2. ObjectDetection retrieves the input images from the instructor-provided S3 bucket.
3. Amazon Rekognition analyzes each image to determine whether it contains a car with confidence greater than 80%.
4. The names of qualifying images are sent to the SQS FIFO queue.
5. EC2-B runs TextRecognition.java and receives image names from the queue.
6. TextRecognition downloads each corresponding image from S3.
7. Amazon Rekognition detects text in each image.
8. Text with confidence greater than 80% is written to output.txt.
9. After all images have been processed, EC2-A sends a termination message (-1) through the queue.
10. EC2-B receives the termination message and stops processing.

## 3. AWS Cloud Environment Setup

### Step 1: Start the AWS Learner Lab

1. Log in to AWS Academy.
2. Open the Learner Lab environment.
3. Start the lab.
4. Open the AWS Management Console.
5. Verify that the AWS region is set to **US East (N. Virginia) – us-east-1**.

### Step 2: Download the SSH PEM Key

Before connecting to the EC2 instances, download the SSH private key provided by AWS Academy Learner Lab.

1. In the Learner Lab environment, locate and open **AWS Details**.
2. Select **Show** to display the AWS details.
3. Locate the option to download the PEM key.
4. Download the PEM file to your local Mac.
5. Keep the file in a location that is easy to access, such as the Downloads folder.

This private key is used to establish SSH connections to both EC2 instances.

For this project, the downloaded key file was named:

`labsuser.pem`

The key must be kept secure and should not be uploaded to GitHub or included in the project submission.

### Step 3: Access the Instructor-Provided S3 Bucket

The project uses an Amazon S3 bucket provided by the instructor. The bucket contains the ten input images required for the application.

The bucket was accessed directly through the Java applications using its provided bucket name. It was not necessary to locate or create the bucket through the S3 console.

**S3 Bucket Name:** `njit-cs-643`

**AWS Region:** `us-east-1`

The bucket name was configured in the Java source files so that the applications could retrieve the input images.

No new S3 bucket was created for this project.

### Step 4: Create the SQS FIFO Queue

1. Open the Amazon SQS service in the AWS Management Console.
2. Select **Create queue**.
3. Choose **FIFO** as the queue type.
4. Enter the queue name:

   `car-detection-queue.fifo`

5. Configure the queue settings as follows:

| Setting | Configuration |
|---|---|
| Queue type | FIFO |
| Content-based deduplication | Enabled |
| Delivery delay | 0 seconds |
| Receive message wait time | 20 seconds |
| Message retention period | 4 days |
| Default visibility timeout | 30 seconds |

6. Create the queue.
7. Copy the queue URL and configure it in the Java applications.

The FIFO queue ensures that messages are processed in the order they are received. Content-based deduplication is enabled to help prevent duplicate messages.

The consumer application also uses a 300-second visibility timeout when receiving messages.

### Step 5: Create the EC2 Instances

Two EC2 instances are required to run the two Java applications.

1. Open the Amazon EC2 service in the AWS Management Console and select **Launch instance**.
2. Enter a name for the first instance, such as **EC2-A**.
3. Under Application and OS Images (AMI), select the Amazon Linux image used for the project. The instances in this project ran Amazon Linux 2023.
4. Select **t2.micro** as the instance type. T2 instances are low-cost, general-purpose instances that provide baseline CPU performance with the ability to burst when needed.
5. Under Key pair, select **vockey**, which should be pre-populated in the Learner Lab environment.
6. Under Network Settings, select **Create security group** and configure the following:
   - Allow SSH traffic from **My IP**.
   - Allow HTTPS traffic from the internet.
   - Allow HTTP traffic from the internet.
7. Leave the Configure Storage and Advanced Details settings at their defaults unless changes are required.
8. Launch the instance.
9. Repeat the process to create the second instance.

The two instances should be named:

- **EC2-A:** Runs ObjectDetection.java.
- **EC2-B:** Runs TextRecognition.java.

### Add IAM Roles to the EC2 Instances

After creating both instances, configure their IAM roles so they can access Amazon S3, SQS, and Rekognition.

1. Open the EC2 Instances page in the AWS Management Console.
2. Verify that both instances are running. If an instance is stopped, select it, choose **Instance state**, and select **Start instance**.
3. Select EC2-A.
4. Navigate to **Actions → Security → Modify IAM role**.
5. Select the pre-populated **LabRole** from the dropdown, if available, and update the IAM role.
6. Repeat the process for EC2-B.

The IAM instance profile must be associated with a role that has permission to access the required AWS services.

## 4. Connect to the EC2 Instances

SSH was used to connect to both EC2 instances from the local Mac.

Before connecting, locate the public IP addresses:

1. Open the Amazon EC2 Instances page.
2. Select EC2-A or EC2-B.
3. Under the **Details** tab, locate the **Public IPv4 address**.
4. Use the corresponding address in the SSH command.

Open the Mac Terminal and connect to EC2-A:

```bash
ssh -i ~/Downloads/labsuser.pem ec2-user@<EC2-A-PUBLIC-IP>
```

Open a separate Mac Terminal window and connect to EC2-B:

```bash
ssh -i ~/Downloads/labsuser.pem ec2-user@<EC2-B-PUBLIC-IP>
```

The public IP addresses may change if the instances are stopped and restarted.

## 5. Install and Verify Java on Both EC2 Instances

Java is required on both EC2 instances to execute the Java applications.

Both instances used Amazon Linux 2023.

1. Connect to EC2-A using SSH.
2. Check whether Java is installed:

   ```bash
   java -version
   ```

3. If Java is not installed, install Amazon Corretto 17:

   ```bash
   sudo dnf install java-17-amazon-corretto -y
   ```

4. Verify the installation:

   ```bash
   java -version
   ```

5. Repeat these steps on EC2-B.

## 6. Create the Maven Project on the Local Mac

The Java source files were developed and compiled on the local Mac before being transferred to the EC2 instances.

The following software was used locally:

- **Java JDK:** Required for Java development and compilation.
- **Apache Maven:** Used to manage project dependencies, compile the Java source files, and package the application into a JAR file.

After installing Maven, verify the installation from the Mac Terminal:

```bash
mvn -version
```

Verify Java:

```bash
java -version
```

### Project Directory Structure

The Maven project was organized as follows:

```text
Project1_CS643/
├── pom.xml
└── src/
    └── main/
        └── java/
            ├── ObjectDetection.java
            └── TextRecognition.java
```

The two Java source files were placed inside the `src/main/java` directory.

## 7. Configure the pom.xml File

The `pom.xml` file is the Maven Project Object Model configuration file. It defines the project information, required dependencies, and build configuration.

The following configurations were included:

- **Project information:** Defines the Maven group ID, artifact ID, and version.
- **AWS SDK dependencies:** Includes the dependencies needed to communicate with Amazon S3, Amazon SQS, and Amazon Rekognition.
- **Maven Shade Plugin:** Packages the application and its dependencies into a single executable JAR file.

The Maven Shade Plugin was configured with `ObjectDetection` as the main application class.

This configuration allowed the Java applications to be compiled and packaged locally, without requiring Maven on the EC2 instances.

## 8. Compile and Package the Java Applications

From the root directory of the Maven project on the local Mac, run:

```bash
mvn clean package
```

This command:

1. Cleans previous build files.
2. Compiles the Java source files.
3. Downloads the required Maven dependencies.
4. Packages the application and its dependencies into a JAR file.

After a successful build, the generated JAR file is located in the project's `target` directory:

```text
target/project1-cs643-1.0-SNAPSHOT.jar
```

To verify that the JAR file was created, run:

```bash
ls -lh target/
```

This displays the files in the target directory, including the generated JAR file and its size.

To display the full path to the JAR file, run:

```bash
pwd
```

The full path is the current project directory followed by:

`/target/project1-cs643-1.0-SNAPSHOT.jar`

The JAR file contains the compiled application and the required AWS SDK dependencies.

## 9. Transfer the JAR File to Both EC2 Instances

After successfully building the project, the JAR file was transferred from the local Mac to both EC2 instances using SCP.

Run these commands from the Mac Terminal, not from inside an EC2 SSH session.

Transfer the JAR file to EC2-A:

```bash
scp -i ~/Downloads/labsuser.pem target/project1-cs643-1.0-SNAPSHOT.jar ec2-user@<EC2-A-PUBLIC-IP>:~
```

Transfer the JAR file to EC2-B:

```bash
scp -i ~/Downloads/labsuser.pem target/project1-cs643-1.0-SNAPSHOT.jar ec2-user@<EC2-B-PUBLIC-IP>:~
```

The JAR file was placed in the home directory of both EC2 instances.

## 10. Run the Applications

Both EC2 instances use the `LabRole` IAM role, allowing the Java applications to access AWS services through the default AWS credentials provider.

The consumer application on EC2-B must be started before the producer application on EC2-A.

### Step 1: Start Text Recognition on EC2-B

Connect to EC2-B:

```bash
ssh -i ~/Downloads/labsuser.pem ec2-user@<EC2-B-PUBLIC-IP>
```

Run the TextRecognition application:

```bash
java -cp ~/project1-cs643-1.0-SNAPSHOT.jar TextRecognition
```

The application begins listening for messages from the SQS FIFO queue.

### Step 2: Start Object Detection on EC2-A

Open a separate Mac Terminal window and connect to EC2-A:

```bash
ssh -i ~/Downloads/labsuser.pem ec2-user@<EC2-A-PUBLIC-IP>
```

Run the ObjectDetection application:

```bash
java -jar ~/project1-cs643-1.0-SNAPSHOT.jar
```

The application retrieves the ten images from the S3 bucket and uses Amazon Rekognition to identify images containing cars.

Images with car detection confidence greater than 80% are sent to the SQS FIFO queue.

After processing all images, the producer sends the termination message `-1`.

EC2-B receives the image messages, processes the images, and stops when it receives the termination message.

## 11. View the Output

After the TextRecognition application finishes processing, the recognized text is saved to:

```text
/home/ec2-user/output.txt
```

To view the results on EC2-B, run:

```bash
cat ~/output.txt
```

The application writes text detected with confidence greater than 80% to the output file.

Images without text above the confidence threshold are not included in the output.

### Results

The car detection application identified seven qualifying images:

- 1.jpg
- 2.jpg
- 3.jpg
- 4.jpg
- 5.jpg
- 6.jpg
- 7.jpg

Images 8.jpg, 9.jpg, and 10.jpg did not contain a car detected above the 80% confidence threshold.

The text recognition application successfully processed the seven qualifying images. The final output file contained recognized text from images 1.jpg, 3.jpg, 4.jpg, and 7.jpg.

Images 2.jpg, 5.jpg, and 6.jpg did not produce text above the 80% confidence threshold.

## 12. Technologies Used

- Java
- Apache Maven
- AWS SDK for Java 2.x
- Amazon EC2
- Amazon S3
- Amazon SQS FIFO
- Amazon Rekognition
- IAM Roles
- Amazon Linux 2023
- Maven Shade Plugin

## 13. Conclusion

This project demonstrates a distributed image recognition application using AWS cloud services.

Two EC2 instances were used to separate car detection and text recognition into producer and consumer applications. Amazon S3 provided the input images, Amazon Rekognition performed image and text analysis, and Amazon SQS FIFO enabled communication between the two applications.

The Java applications were developed and packaged locally using Maven, and the resulting executable JAR file was transferred to both EC2 instances for execution.

The application successfully identified images containing cars, processed the qualifying images for text, and saved the recognized results to an output file.

In this project, AI tools such as Github Co-pilot and ChatGPT were helpful in troubleshooting bugs related to code in either Java application, as well as understanding the EC2 instance and SQS queue configuration setup better. 
