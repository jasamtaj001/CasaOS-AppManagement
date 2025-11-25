# **Manual Update Guide for CasaOS-AppManagement**

This guide assumes you are logged into your Ubuntu 24.04 server via SSH and are currently in your user's home directory.

## **Step 1: Install Build Dependencies**

To compile the code, you need **Go (Golang)** and **Git**. Ubuntu 24.04 includes a recent version of Go in its default repositories that is compatible with CasaOS.

Run the following command to install them:

sudo apt update  
sudo apt install \-y golang-go git

**Verify installation:**

go version  
\# Output should be something like: go version go1.22.0 linux/amd64

## **Step 2: Clone Your Modified Repository**

You need to download the source code from your GitHub repository where you applied the fix.

*Replace \<YOUR\_GITHUB\_REPO\_URL\> with the actual link to your repository.*

\# Clone the repository  
git clone \<YOUR\_GITHUB\_REPO\_URL\>

\# Enter the directory  
cd CasaOS-AppManagement

## **Step 3: Install Generators and Generate Code**

The missing packages (codegen/message\_bus, etc.) are created by code generators. You must install these tools and run the generation command.

1. **Add Go Binaries to Path** (Temporary session fix):  
   export PATH=$PATH:$(go env GOPATH)/bin

2. Install the Code Generator Tools:  
   Note: We use specific versions compatible with CasaOS.  
   \# Install oapi-codegen (likely used for API generation)  
   go install \[github.com/deepmap/oapi-codegen/cmd/oapi-codegen@latest\](https://github.com/deepmap/oapi-codegen/cmd/oapi-codegen@latest)

   \# Install mockgen (used for testing/mocking interfaces)  
   go install \[github.com/golang/mock/mockgen@latest\](https://github.com/golang/mock/mockgen@latest)

3. Generate the Missing Code:  
   This command looks for //go:generate directives in the source code and creates the missing files.  
   go generate ./...

4. Tidy Modules:  
   Ensure all dependencies are synced.  
   go mod tidy

## **Step 4: Compile the Fixed Binary**

Now that the generated files exist, build the binary.

**For standard x86\_64/AMD64 (Most PCs/Servers):**

go build \-v \-o casaos-app-management

For ARM64 (Raspberry Pi 4/5, ZimaBoard):  
Only use this if your uname \-m output is aarch64.  
env GOOS=linux GOARCH=arm64 go build \-v \-o casaos-app-management

Check the result:  
Run ls \-l to confirm a file named casaos-app-management was created.

### **Important Note on Configuration**

Your investigation using command  
	systemctl cat casaos-app-management | grep ExecStart  
confirmed the systemd service expects the binary at /usr/bin/casaos-app-management and looks for its configuration file at /etc/casaos/app-management.conf. Since we are only replacing the executable, the service will automatically use the existing configuration file, which is exactly what we want.

## **Step 5: Backup and Replace the System Binary**

We will now stop the running service, back up the original file (for safety), and install your new one.

1. **Stop the Service**  
   sudo systemctl stop casaos-app-management

2. **Backup Original Binary**  
   \# Move the existing binary to a backup file  
   sudo mv /usr/bin/casaos-app-management /usr/bin/casaos-app-management.bak

3. **Install New Binary**  
   \# Copy your newly compiled binary to the system folder  
   sudo cp casaos-app-management /usr/bin/

   \# Ensure it is executable  
   sudo chmod \+x /usr/bin/casaos-app-management

## **Step 6: Restart and Verify**

Now that the binary is replaced, restart the service and check if the error is resolved.

1. **Start the Service**  
   sudo systemctl start casaos-app-management

2. **Check Service Status**  
   sudo systemctl status casaos-app-management

   *You should see Active: active (running).*  
3. Monitor Logs (Crucial)  
   Watch the logs in real-time to ensure the Docker API error is gone.  
   sudo journalctl \-u casaos-app-management \-f

   *Trigger the action that previously caused the crash (e.g., opening the App Store). If the logs remain clean, your fix worked.*

## **Troubleshooting**

If something goes wrong and the service fails to start, you can revert to the original binary using these commands:

sudo systemctl stop casaos-app-management  
sudo mv /usr/bin/casaos-app-management.bak /usr/bin/casaos-app-management  
sudo systemctl start casaos-app-management  
