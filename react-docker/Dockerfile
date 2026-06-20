FROM node:26-alpine

#`RUN addgroup app && adduser -S -G app app` is used in a Dockerfile to create a new user group and a new user within the Docker container. Here's a breakdown of what each part does:
#why we did this explanation:
#1. `addgroup app`: This command creates a new user group named "app"
#2. `adduser -S -G app app`: This command creates a new user named "app" and adds them to the "app" group n
RUN addgroup app && adduser -S -G app app

#set the user to run the application as the newly created user "app" instead of the default root user. This is a security best practice to minimize potential risks associated with running applications as the root user.
USER app

#set the working directory inside the container to /app. This means that any subsequent commands (like COPY, RUN, CMD) will be executed relative to this directory.
WORKDIR /app

#copy package.json and package-lock.json to the working directory. This is done to install dependencies before copying the entire application code, which can help leverage Docker's caching mechanism and speed up builds.
COPY package*.json ./

#sometimes the owneship of the files int the working directory is changed to root and thus the app can't accesss the files. To avoid this, we change the ownership of the files to the "app" user and group.

USER root

#Change the ownership of the files in the working directory to the "app" user and group. This ensures that the "app" user has the necessary permissions to access and modify these files.
RUN chown -R app:app /app

#switch back to the "app" user to continue running subsequent commands as this user.
USER app

#install the application dependencies defined in package.json. This command will read the package.json file and install the required Node.js packages into the container.
RUN npm install

#Copy the entire application code from the host machine to the working directory inside the container. This includes all files and directories in the current context.
COPY . .

#expose port 5173 to allow external access to the application running inside the container. This is useful for networking and communication with other containers or services.
EXPOSE 5173

#command to start the application when the container is run. In this case, it runs the command `npm run dev`, which typically starts a development server for a Node.js application.
CMD ["npm", "run", "dev"]
