# day:6 file-io-practice.mdStep 
Step 1: Create File
command:touch notes.txt
Purpose:
Creates an empty file named notes.txt

Step 2: Write First Line (Overwrite)
command: echo "line1">note.txt
Purpose:
Writes “Line 1” into the file.
If file already has content, it will be removed.

Step 3: Append Second Line
command:echo "line 2" >> notes.txt
Purpose:
Adds “Line 2” at the end of the file without deleting existing data.

Step 4: Write and Display Using tee
command:echo "line 3" | tee -a notes.txt
Purpose:
Displays “Line 3” on screen and appends it to the file at the same time.

Step 5: View Full File Content
command:cat notes.txt
Purpose:
Shows the complete content of the file.

Step 6: View First Two Lines
command:head -n 2 notes.txt
Purpose:
Displays the first two lines of the file.

Step 7: View Last Two Lines
command:tail-n 2 notes.txt
Purpose:
Displays the last two lines of the file.[Linux Read and write Text files-Day-6.pdf](https://github.com/user-attachments/files/25418811/Linux.Read.and.write.Text.files-Day-6.pdf)
