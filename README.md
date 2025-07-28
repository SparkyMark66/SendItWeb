# **SendIt Web Setup Guide for Apache2**

This guide provides all the necessary code and instructions to set up a web-based front-end for the Data Sharer application on an Apache2 server. This allows any device on the local network to send messages via a web browser.

This guide assumes you have a running Apache2 server on a Debian-based Linux system (like Ubuntu or Raspberry Pi OS).

## **Part 1: The Frontend Web Page (index.html)**

This is the HTML file that creates the user interface in the web browser. It contains the form for typing the target address and the message, and it uses JavaScript to send this data to the backend script.

### **Code for index.html**

Create a file named index.html with the following content:

\<\!DOCTYPE html\>  
\<html lang="en"\>  
\<head\>  
    \<meta charset="UTF-8"\>  
    \<meta name="viewport" content="width=device-width, initial-scale=1.0"\>  
    \<title\>Web Data Sender\</title\>  
    \<script src="https://cdn.tailwindcss.com"\>\</script\>  
    \<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700\&display=swap" rel="stylesheet"\>  
    \<style\>  
        body {  
            font-family: 'Inter', sans-serif;  
        }  
    \</style\>  
\</head\>  
\<body class="bg-gray-100 flex items-center justify-center min-h-screen"\>

    \<div class="w-full max-w-2xl mx-auto bg-white rounded-xl shadow-lg p-8"\>  
        \<div class="mb-6"\>  
            \<h1 class="text-3xl font-bold text-gray-800"\>Web Data Sender\</h1\>  
            \<p class="text-gray-500 mt-1"\>Send a message to any computer on the local network.\</p\>  
        \</div\>

        \<\!-- Form for sending data \--\>  
        \<form id="sendForm"\>  
            \<div class="mb-4"\>  
                \<label for="target\_address" class="block text-sm font-medium text-gray-700 mb-1"\>Target IP / Hostname\</label\>  
                \<input type="text" id="target\_address" name="target\_address" value="\<broadcast\>" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition"\>  
                \<p class="text-xs text-gray-500 mt-1"\>Use \`\<broadcast\>\` to send to all devices, or a specific IP (e.g., 192.168.1.50).\</p\>  
            \</div\>

            \<div class="mb-6"\>  
                \<label for="message" class="block text-sm font-medium text-gray-700 mb-1"\>Message\</label\>  
                \<textarea id="message" name="message" rows="8" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition" placeholder="Type your message here..."\>\</textarea\>  
            \</div\>

            \<div class="flex items-center justify-between"\>  
                \<button type="submit" id="sendButton" class="w-full bg-blue-600 text-white font-bold py-3 px-6 rounded-lg hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 transition-transform transform hover:scale-105"\>  
                    Send Message  
                \</button\>  
            \</div\>  
        \</form\>

        \<\!-- Status message area \--\>  
        \<div id="statusMessage" class="mt-6 text-center p-4 rounded-lg text-sm" role="alert"\>\</div\>  
    \</div\>

    \<script\>  
        const sendForm \= document.getElementById('sendForm');  
        const sendButton \= document.getElementById('sendButton');  
        const statusMessage \= document.getElementById('statusMessage');

        sendForm.addEventListener('submit', async (e) \=\> {  
            e.preventDefault(); // Prevent the default form submission

            const targetAddress \= document.getElementById('target\_address').value.trim();  
            const message \= document.getElementById('message').value.trim();

            if (\!targetAddress || \!message) {  
                showStatus('Please fill out all fields.', 'error');  
                return;  
            }  
              
            sendButton.disabled \= true;  
            sendButton.textContent \= 'Sending...';  
            sendButton.classList.add('opacity-50', 'cursor-not-allowed');

            try {  
                // The URL to your backend Python CGI script  
                const response \= await fetch('/cgi-bin/send\_udp.py', {  
                    method: 'POST',  
                    headers: {  
                        'Content-Type': 'application/json',  
                    },  
                    body: JSON.stringify({  
                        target\_address: targetAddress,  
                        message: message,  
                    }),  
                });

                const result \= await response.json();

                if (result.status \=== 'success') {  
                    showStatus(\`Message successfully sent to ${targetAddress}\!\`, 'success');  
                    document.getElementById('message').value \= ''; // Clear message on success  
                } else {  
                    showStatus(\`Error: ${result.message}\`, 'error');  
                }

            } catch (error) {  
                showStatus('A network or server error occurred. Make sure the backend script is set up correctly.', 'error');  
                console.error('Fetch Error:', error);  
            } finally {  
                sendButton.disabled \= false;  
                sendButton.textContent \= 'Send Message';  
                sendButton.classList.remove('opacity-50', 'cursor-not-allowed');  
            }  
        });

        function showStatus(message, type) {  
            statusMessage.textContent \= message;  
            if (type \=== 'success') {  
                statusMessage.className \= 'mt-6 text-center p-4 rounded-lg text-sm bg-green-100 text-green-800';  
            } else {  
                statusMessage.className \= 'mt-6 text-center p-4 rounded-lg text-sm bg-red-100 text-red-800';  
            }  
        }  
    \</script\>

\</body\>  
\</html\>

### **Placement**

Place this index.html file in your web server's root directory, which is typically /var/www/html/.

sudo mv index.html /var/www/html/

## **Part 2: The Python Backend Script (send\_udp.py)**

This Python script acts as a CGI (Common Gateway Interface) script. Apache will execute this script whenever the web frontend sends it a request. The script reads the data from the request, opens a UDP socket, and sends the message over the network.

### **Code for send\_udp.py**

Create a file named send\_udp.py with the following content.

\#\!/usr/bin/env python3  
\# \-\*- coding: UTF-8 \-\*-

import cgi  
import cgitb  
import json  
import socket  
import sys

\# Enable CGI traceback for debugging  
cgitb.enable()

\# \--- Configuration \---  
UDP\_PORT \= 61991  
ENCODING \= 'utf-8'

def send\_response(data):  
    """Prints the JSON response with the correct header."""  
    print("Content-Type: application/json")  
    print("Access-Control-Allow-Origin: \*") \# Allow requests from any origin  
    print() \# End of headers  
    print(json.dumps(data))  
    sys.exit()

def main():  
    """Main function to handle the POST request and send UDP packet."""  
    try:  
        \# Read the raw POST data from standard input  
        post\_data \= sys.stdin.read()  
        if not post\_data:  
            send\_response({"status": "error", "message": "No data received in POST request."})

        \# Parse the JSON data  
        data \= json.loads(post\_data)  
        target\_addr \= data.get('target\_address')  
        message \= data.get('message')

        if not target\_addr or not message:  
            send\_response({"status": "error", "message": "Missing 'target\_address' or 'message' in request."})

    except json.JSONDecodeError:  
        send\_response({"status": "error", "message": "Invalid JSON format in request body."})  
        return  
    except Exception as e:  
        send\_response({"status": "error", "message": f"Error processing request: {e}"})  
        return

    \# Now, send the UDP packet  
    try:  
        \# Create a UDP socket  
        sock \= socket.socket(socket.AF\_INET, socket.SOCK\_DGRAM)  
        \# Set socket options to allow broadcasting  
        sock.setsockopt(socket.SOL\_SOCKET, socket.SO\_BROADCAST, 1\)

        \# Send the message  
        sock.sendto(message.encode(ENCODING), (target\_addr, UDP\_PORT))  
        sock.close()

        \# Send a success response back to the web client  
        send\_response({"status": "success", "message": "Data sent successfully."})

    except socket.gaierror:  
        send\_response({"status": "error", "message": f"Hostname could not be resolved: {target\_addr}"})  
    except Exception as e:  
        send\_response({"status": "error", "message": f"Failed to send UDP packet: {e}"})

if \_\_name\_\_ \== "\_\_main\_\_":  
    main()

### **Placement**

Place this send\_udp.py file in Apache's CGI directory, which is typically /usr/lib/cgi-bin/.

sudo mv send\_udp.py /usr/lib/cgi-bin/

## **Part 3: Server Configuration**

After placing the files, you need to configure Apache to execute the Python script.

### **Step 1: Make the Python Script Executable**

The server needs permission to run the script. Use the chmod command to grant execute permissions.

sudo chmod \+x /usr/lib/cgi-bin/send\_udp.py

### **Step 2: Enable the CGI Module**

Apache needs the cgi module to be enabled to handle CGI scripts.

sudo a2enmod cgi

### **Step 3: Restart Apache**

For all the changes to take effect, you must restart the Apache service.

sudo systemctl restart apache2

## **Part 4: Testing**

You are now ready to go\!

1. Open a web browser on any device on your local network.  
2. Navigate to your server's IP address (e.g., http://192.168.1.100). You should see the "Web Data Sender" page.  
3. Make sure you have the Python desktop application running on another computer to receive the message.  
4. Fill out the form on the webpage and click "Send Message". The message should appear in the desktop client, showing it was sent from your server's IP address.
