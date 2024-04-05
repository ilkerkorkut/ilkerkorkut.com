---
title: Efficiently Finding PIDs by Binary Name in Linux Using Go
author: ilker
date: 2024-03-06
tags:
  - os
  - linux
  - go
  - golang
categories:
  - go
keywords:
  - go
  - linux
---

For many system functions, managing processes and finding their Process IDs (PIDs) is essential. However, it can occasionally be difficult to retrieve PIDs based on the binary name, particularly if the names are lengthy or are not presented correctly when using tools like (process status) `ps`. We will now examine a Go function that effectively locates Linux OS process IDs using their binary names.

#### Issue: Finding Processes Using Binary Names

It is necessary to be able to identify individual processes using their binary names while working with a heterogeneous group of processes on a Linux system. Linux systems automatically truncate the process name to make it fit into a process listing field with a specified width.
Many system utilities, such as `ps`, `top`, and `htop`, frequently exhibit the process name truncation. These tools frequently truncate the representation by shortening the command line inputs to the width of the process name.

#### Solution: Construct Function to Retrieve Process IDs

The Go function that follows provides a methodical way to find processes based on their binary names. Let's dissect the main elements of the role:

```go
const (
	procPath = "/proc"
	exePath  = "exe"
)

func findPID(binaryName string) ([]int, error) {
	processIDs := make([]int, 0)

	procs, err := os.ReadDir(procPath)
	if err != nil {
		return nil, err
	}

	for _, proc := range procs {
		pid, err := strconv.Atoi(proc.Name())
		if err != nil {
			continue
		}

		binPath := filepath.Join(procPath, proc.Name(), exePath)

		link, err := os.Readlink(binPath)
		if err != nil {
			continue
		}

		fileName := filepath.Base(link)

		if strings.EqualFold(fileName, binaryName) {
			processIDs = append(processIDs, pid)
		}
	}

	return processIDs, nil
}
```

1. *Reading `/proc` Directory*: The function starts by reading the `/proc` file system to obtain a list of directories representing active processes. The function reads the list of directories in the `/proc` file system and iterates through each directory that represents a process. It constructs the path to the binary file `/proc/<PID>/exe` and reads the symbolic link using `os.Readlink`. Then, it compares the file name obtained from the symbolic link to the desired `binaryName` using a case-insensitive comparison.
2. *Iterating and Matching*: It iterates through each directory, extracting the PID and the path to the binary file of the process.
3. *Comparing Binary Names*: By reading the symbolic link of the binary file path and extracting the file name, it compares the binary name with the desired name in a case-insensitive manner.
4. *Building Result*: When a match is found, the function adds the corresponding PID to the list of process IDs.

Benefits and Applications
* Precision in Process Identification: This function ensures accurate mapping of processes to their binary names, overcoming the limitations of standard tools.
* Process Monitoring and Management: Ideal for monitoring specific processes or managing them based on the binary name.
* Improved System Efficiency: Enhances system efficiency by enabling targeted actions on processes.