# Port Allocator Web App

A user-friendly web application for tracking and managing port allocations for Docker containers, services, and stacks across hosts and clusters.

![Port Allocator Web App](https://placekitten.com/800/400)

## Overview

Port Allocator helps system administrators track which ports are being used by Docker containers and services in their environments. It allows you to:

* Generate unique port numbers for new containers and services
* Organize containers into stacks and services
* Group hosts into clusters
* Keep track of internal and external exposure points
* Manage environment variables, networks, and volumes

## 🚀 Quick Start Guide for Administrators

This guide will help you deploy the Port Allocator web app even if you're not familiar with Docker or Vue.js applications.

### Prerequisites

* Docker and Docker Compose installed on your system
* Basic familiarity with command line operations

### Deployment Steps

1. **Get the application code**

   Clone this repository or download it as a zip file and extract it to a directory on your server.

   ```bash
   git clone https://github.com/yourusername/portgen-webui-base.git
   cd portgen-webui-base
   ```

2. **Run the application in production mode**

   From the application directory, run:

   ```bash
   docker-compose up -d app-prod
   ```

   This command builds the application and starts it in a container.

3. **Access the web application**

   Open your browser and navigate to:

   ```
   http://localhost:8890
   ```

   If you're accessing it from another machine, replace "localhost" with your server's IP address or hostname.

4. **Stop the application**

   When you want to stop the application, run:

   ```bash
   docker-compose down
   ```

### Customizing the Configuration

#### Changing the Port

If you need to run the application on a different port (not 8890), edit the `docker-compose.yml` file:

1. Open `docker-compose.yml` in a text editor
2. Find the line `- "8890:80"` under the `app-prod` service
3. Change `8890` to your desired port number
4. Save the file and restart the application

Example:
```yaml
# Change this:
ports:
  - "8890:80"

# To this (for port 9000):
ports:
  - "9000:80"
```

## ⚠️ Important Notes About Data Persistence

**The current version uses browser localStorage for data storage. This means:**

1. **Data is stored in the browser**, not on the server
   * If you clear your browser data, you will lose all your port allocation information
   * Different browsers/computers will not share the same data

2. **No multi-user support**
   * Each user accessing the application will have their own separate data
   * There is no centralized database or user authentication

3. **No automatic backup**
   * We recommend taking screenshots or regularly exporting important information

### Planned Future Improvements

We are planning to add proper server-side storage in a future version, which will address these limitations by:

* Storing data in a central database
* Supporting multiple users with authentication
* Providing backup and restoration capabilities
* Enabling shared access to port allocation data

## 🛠️ Usage Guide

### Creating a New Entry

1. Select the type of entry you want to create (host, cluster, container, service, or stack)
2. Click the "Create" button
3. Fill in the required information
4. For containers and services, click "Generate Port" to get a unique port number
5. Click "Save New Entry"

### Managing Entries

Use the tabs at the bottom to switch between:
* **Saved Entries**: View all your created entries
* **Hosts**: View and manage host entries
* **Clusters**: View and manage cluster entries

### Editing or Deleting Entries

In the "Saved Entries" tab:
1. Find the entry you want to modify
2. Click "Edit" to update the entry or "Delete" to remove it

### Dark/Light Mode

Toggle between dark and light mode using the theme button at the bottom-right corner of the page.

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🙋 Support

If you encounter any issues or have questions, please file an issue in the repository or contact your system administrator.