# Visual development with VSCode

This article aims to showcase a visual development (in a web browser) with a source control sync using VSCode.

# Setup

- DEV HCC environment
- One DEV namespace (equals the `main` branch in source control)
- One USER namespace per developer (`elebedyu` in this article)
- Developers have the credentials to access the DEV environment
- Each developer has VSCode configured to connect to his own namespace
- [Repo](https://scm.dev.isccloud.io/eduard.lebedyuk/visual)

This article does not cover CICD, only the development part of the process.

# Process

1. Create a [new issue](https://scm.dev.isccloud.io/eduard.lebedyuk/visual/-/issues/new)

![image](https://user-images.githubusercontent.com/5127457/205972194-3abc0c91-c36e-41e3-87ef-93d0af2d40c5.png)

Issues contain various information: description, due date, discussion, assigned developer, current state, and so on.

2. Create a branch for your work and a merge request from the newly created Issue page. The branch must be sourced from the `main` branch (which is your `dev` environment). This branch is a feature branch - it exists solely to develop one feature.

![image](https://user-images.githubusercontent.com/5127457/205972539-008bca1b-02df-4279-9f2c-bdc29e7d26ca.png)

3. Go to VSCode and checkout the feature branch (`1-add-production` in my case):

![image](https://user-images.githubusercontent.com/5127457/205973358-424674e6-ad3f-42b4-857d-6c446d85b418.png)

4. At this point, the feature branch equals the `main` branch (and the `dev` environment). The developer's environment must reflect that. RMB the `src` folder and choose `Import and Compile`. 

![image](https://user-images.githubusercontent.com/5127457/205974218-4fe466ee-b38f-45ad-8318-8e796b0a5ca9.png)

It will overwrite the developer's namespace with the current state of the `dev` environment

5. Develop the feature using any tool of your choice.

![image](https://user-images.githubusercontent.com/5127457/205976778-4c563825-e344-4b00-ad9f-6f5f5e4140ec.png)

6. Go to the VSCode, ObjectScript tab and export either everything or just the classes you modified.

![image](https://user-images.githubusercontent.com/5127457/205979304-36d25702-24bb-45b0-90da-499658c3bcd1.png)

7. Go to the Source Control tab and stage individual items you modified (by pressing a plus to the right of them) or stage everything by pressing a plus to the right of changes.

 ![image](https://user-images.githubusercontent.com/5127457/205979667-369b6d7c-a8e0-4e1c-b38e-9187f6cbfe0e.png)
 
 8. Write a commit message describing your changes and press the Commit button.

![image](https://user-images.githubusercontent.com/5127457/205980178-b8803963-7271-4458-abfe-5c4ebcdab08d.png)

9. Press the Sync button to send your changes to the server.

![image](https://user-images.githubusercontent.com/5127457/205980664-99a82535-c91e-460a-9a14-6806995c08df.png)

Repeat steps 5-9 till your feature is complete.

10. Go to your merge request and review the changes.

![image](https://user-images.githubusercontent.com/5127457/205981066-b215f892-abe7-4543-88fc-df4ab25773fc.png)

11. Mark the merge request as ready and merge it:

![image](https://user-images.githubusercontent.com/5127457/205981216-f9fe5d89-3a5f-4b05-8adb-90fc6c9d8a9f.png)

Note that it also closes the corresponding issue.

12. Check that the pipeline succeeded:

![image](https://user-images.githubusercontent.com/5127457/205982274-a3be5a5f-540c-4906-8655-c3dd87502747.png)

Go to pipeline logs to validate that CICD run was completed successfully.

13. Go to the DEV namespace (using a browser or VSCode) to check that the changes are there. 
