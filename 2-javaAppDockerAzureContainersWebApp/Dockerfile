# Use an official Python runtime as a parent image
FROM python

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Run python's package manager and install the flask package
RUN pip install flask

# Configure ports
EXPOSE 2222 80

# Run apt-get, to install the SSH server
RUN apt-get update \ 
    && apt-get install -y --no-install-recommends openssh-server \
    && echo "root:Docker!" | chpasswd

#Copy the sshd_config file to its new location
COPY sshd_config /etc/ssh/

# Start the SSH service
RUN service ssh start

# Copy init_container.sh to the /bin directory
COPY init_container.sh /bin/

# Run the chmod command to change permissions on above file in the /bin directory
RUN chmod 755 /bin/init_container.sh 

# run commands in init_container.sh
CMD ["/bin/init_container.sh"]