// include and definitions
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
// to check whether character is letter or digit
#include <ctype.h>
#include <sys/wait.h>
#include<fcntl.h>

// get value function
// Helper Method to get value from environ 
char *get_value(char ** arr, const char *name, int name_length){
    int count = 0;
    while(arr[count] != NULL){
        // strncmp compares only n characters given in third argument
        // checking if name matches and name stops there
        if(strncmp(arr[count], name, name_length) == 0 && arr[count][name_length] == '='){
            // skips name and =
            return arr[count] + name_length + 1;
        }
        count ++;
    }
    return NULL;
}

// Helper method to check for redirection
int is_redirect(const char *s){
	return strcmp(s, ">") == 0 || strcmp(s, ">>") == 0 || strcmp(s, "]") == 0 || strcmp(s, "]]") == 0;
}
// Expand Token Function
// Helper Function 
char *expand_token(char **env, const char *token){
	// copying all input from token vector
	int capacity = strlen(token) + 1;
	int token_len = strlen(token); 
	char * out = malloc(capacity);
	if(out == NULL){
		perror("malloc");
			exit(1);
	}
	int i = 0;
	int j = 0;
	while(token[i] != '\0'){
		// characters with $ are environmental variables
		if(token[i] == '$'){
			int name_len = 0;
			// only taking alphanumeric values and underscore
			while(isalnum(token[i + 1 + name_len]) || token[i + 1 + name_len] == '_'){
				name_len += 1;
			}
			// getting the value from environ
			char *value = get_value(env, token + i + 1, name_len);
			if(value != NULL){
				int value_len = strlen(value);
				int needed = j + value_len + (token_len - i - 1 - name_len) + 1;
				if(needed > capacity){
					capacity = 2 * needed;
					out = realloc(out, capacity);
					if(out == NULL){
						perror("realloc");
						exit(1);
					}
				}
				for(int k = 0; k < strlen(value); k++){
					out[j] = value[k];
					j++;
				}
			}
			// doesnt say what to do so skipped
			i += 1 + name_len;
		}
				 
		// normal characters should be copied as it is
		else{
			out[j] = token[i];
			i++;
			j++;
		}
	}
	out[j] = 0;
	return out;
}
	

int main(int argc, char ** argv){
	// getting environment
	extern char ** environ;
	// getting argument for getline()
	char *line = NULL;
	size_t len = 0;
	// for storing after strtok
	char **tokens;
	int capacity = 30;
	tokens = malloc(capacity * sizeof(char*));
	if(tokens == NULL){
		perror("malloc");
		exit(1);
	}
	// Creating my own environment table
	int e_count = 0;
	// getting count of environ
	while(environ[e_count] != NULL){
		e_count++;
	}
	char **my_env;
	int env_capacity = e_count + 32;
	my_env = malloc(env_capacity * sizeof(char*));
	if(my_env == NULL){
		perror("malloc");
		exit(1);
	}
	// Transferring to my Table
	for(int i = 0; i < e_count; i++){
		//strdup makes private copy on the heap to chew on
		my_env[i] = strdup(environ[i]);
		if(my_env[i] == NULL){
			perror("strdup");
			exit(1);
		}
	}
	// To know when table terminates
	my_env[e_count] = NULL;
	// String arrays for alias
	char ** alias_names;
	char ** alias_values;
	int alias_count = 0;
	int alias_capacity = 16;
	alias_names = malloc(alias_capacity * sizeof(char*));
	alias_values = malloc(alias_capacity * sizeof(char*));
	if(alias_names == NULL){
		perror("malloc");
		exit(1);
	}
	if(alias_values == NULL){
		perror("malloc");
		exit(1);
	}
	// Calling getline()
	// Exiting if exit or Ctrl-D
	while(1){
		printf("myshell> ");
		fflush(stdout);
		//Takes input till enter
		// getline automatically adds \n
		ssize_t argument = getline(&line, &len, stdin);
		// Ctrl - D gives -1
		if(argument == -1){
			printf("\n");
			break;
		}
		// Token with strtok
		int count = 0;
		//Parsing for "\n", " ", "\t" 
		char *token = strtok(line," \t\n");
		while(token != NULL){
			if(count + 1 >= capacity){
				capacity *= 2;
				tokens = realloc(tokens, capacity * sizeof(char*));	
				if(tokens == NULL){
					perror("realloc");
					exit(1);
				}
			}		
			// storing each token in args array
			tokens[count] = token;
			count++;
			token = strtok(NULL, " \t\n");
			}
		// For iterating
		tokens[count] = NULL;	
		if(count == 0){
			continue;
		}
		// Expanding tokens with helper methods
		for(int i = 0; i < count; i++){
			tokens[i] = expand_token(my_env, tokens[i]);
		}
		// keeping track of times which alias is expanded to avoid repetition
		int *used = calloc(alias_count + 1, sizeof(int));
		if(used == NULL){
			perror("calloc");
			exit(1);
		}
		while(1){
			// finding if token[0] is an alias
			int idx = -1;
			for(int k = 0; k < alias_count; k++){
				if(strcmp(alias_names[k], tokens[0]) == 0){
					idx = k;
					break;
				}
			}
			// stopping if no alias or already expanded
			if(idx == -1 || used[idx]){
				break;
			}
			used[idx] = 1;
			char *copy = strdup(alias_values[idx]);
			if(copy == NULL){
				perror("strdup");
				exit(1);
			}
			// splitting alias command
			int max_pieces = strlen(copy) + 1;
			char **pieces = malloc(max_pieces * sizeof(char*));
			if(pieces == NULL){
				perror("malloc");
				exit(1);
			}
			int n = 0;
			char *p = strtok(copy, " \t\n");
			while(p != NULL){
				pieces[n] = strdup(p);
				if(pieces[n] == NULL){
					perror("strdup");
					exit(1);
				}
				n++;
				p = strtok(NULL, " \t\n");
			}
			free(copy);
			// replacing old values with new
			int new_count = n + count -1;
			// resizing if needed
			if(new_count + 1 >= capacity){
				capacity = 2 * (new_count + 1);
				tokens = realloc(tokens, capacity * sizeof(char*));
				if(tokens == NULL){
					perror("realloc");
					exit(1);
				}
			}
			//  freeing old command
			free(tokens[0]);
			// moving remaining arguments
			if(n > 1){
				for(int i = count - 1; i >= 1; i --){
					tokens[i + n - 1] = tokens[i];
				}
			}
			else if(n == 0){
				for(int i = 1; i < count; i++){
					tokens[i - 1] = tokens[i];
				}
			}
			// Putting in new values;
			for(int i = 0; i < n; i++){
				tokens[i] = pieces[i];
			}
			free(pieces);
			count = new_count;
			tokens[count] = NULL;
			// alias command was empty
			if(count == 0){
				break;
			}
		}
		free(used);
		if(count == 0){
			continue;
		}
		// If command is to exit
		int should_exit = 0;
		if(strcmp(tokens[0], "exit") == 0){
			should_exit = 1;
		}
		// Handling pwd
		else if(strcmp(tokens[0], "pwd") == 0){
			char cwd[4096];
			// getting current working directory
			if(getcwd(cwd, sizeof(cwd))!= NULL){
				printf("%s\n", cwd);
			}
			else{
				perror("pwd");
			}
		}
		// Executing a cd command
		else if(strcmp(tokens[0], "cd") == 0){
			// using default case if not mentioned
			char * dir;
			if(tokens[1] == NULL){
				dir = get_value(my_env, "HOME", 4);
			}
			// taking directory from cla
			else{
				dir = tokens[1];
			}
			// checking if home is not set
			if(dir == NULL){
				fprintf(stderr, "cd: HOME not set\n");
			}
			// calling chdir
			else if(chdir(dir) == -1){
				perror("cd");
			}
		}
		//Executing env command
		else if(strcmp(tokens[0], "env") == 0){
			// Printing all lines
			for(int i = 0; my_env[i] != NULL; i++){
				printf("%s\n", my_env[i]);
			}
		}
		// Building setenv command
		// The setenv command helps the user to create his own variables in environment variables
		else if(strcmp(tokens[0], "setenv") == 0){
			if(tokens[1] == NULL){
				fprintf(stderr, "setenv needs a variable name\n");
			}
			else{
				// length for name, = and \0
				int total = strlen(tokens[1]) + 1 + 1;
				//getting total length
				for(int i = 2; tokens[i] != NULL; i++){
					total += strlen(tokens[i]) + 1;// for space	
				}
				char * prompt = malloc(total);
				if(prompt == NULL){
					perror("malloc");
					exit(1);
				}
				strcpy(prompt, tokens[1]);
				strcat(prompt, "=");
				// joining token values
				for(int j = 2; tokens[j] != NULL; j++){
					if(j > 2){
						strcat(prompt, " ");
					}
					strcat(prompt, tokens[j]);
				}
				// Checking if variable name already exists
				int var_length = strlen(tokens[1]);
				int found = -1;
				//finding where found or not found
				for(int i = 0; my_env[i] != NULL; i++){
					if(strncmp(my_env[i], tokens[1], var_length) == 0 && my_env[i][var_length] == '='){
						found = i;
						break;
					}
				}
				// replacing value
				if(found != -1){
					free(my_env[found]);
					my_env[found] = prompt;
				}
				else{
					// resizing if needed
					if(e_count + 1 >= env_capacity){
						env_capacity *= 2;
						my_env = realloc(my_env, env_capacity * sizeof(char*));
						if(my_env == NULL){
							perror("realloc");
							exit(1);
						}
					}
					my_env[e_count] = prompt;
					e_count++;
					my_env[e_count] = NULL;
				}
			}
		}
		// Executing unsetenv command
		// removes the mentioned variable
		else if(strcmp(tokens[0], "unsetenv") == 0){
			if(tokens[1] == NULL){
				fprintf(stderr, "unsetenv needs a variable name\n");
			}
			// finding the entry like above
			else{
				int length = strlen(tokens[1]);
				int found = -1;
				for( int i = 0; my_env[i] != NULL; i++){
					if(strncmp(my_env[i], tokens[1], length) == 0 && my_env[i][length] == '='){
						found = i;
						break;
					}
				}
				// if found removing or not doing anything 
				if(found != -1){
					free(my_env[found]);
					// shifting every element 1 down
					for(int i = found; my_env[i] != NULL; i++){
						my_env[i] = my_env[i+1];
					}
					e_count --;
				}
			}
		}
		// Writing the alias command
		// The alias helps running commands 
		// Typing the token[1] in command prompt is equivalent to typing the next tokens given	
		else if(strcmp(tokens[0], "alias") == 0){
			// If no first argument, then print current aliases
			if(tokens[1] == NULL){
				for(int i = 0; i < alias_count; i++){
					printf("%s = %s\n", alias_names[i], alias_values[i]);
				}
			}
			else if(tokens[2] == NULL){
				fprintf(stderr, "alias needs a command\n");
			}
			else{
				int total = 1;
				// getting amount of space needed
				for(int i = 2; tokens[i] != NULL; i++){
					total += strlen(tokens[i]) + 1;
				}
				// same as strenv
				char * command = malloc(total);
				if(command == NULL){
					perror("malloc");	
					exit(1);
				}
				// making it so strcat can append
				command[0] = '\0';
				for(int j = 2; tokens[j] != NULL; j++){
					if(j>2){
							strcat(command, " ");
					}
					strcat(command, tokens[j]);
				}
				// checking if the alias already exists
				int found = -1;
				for(int k = 0; k < alias_count; k++){
					if(strcmp(alias_names[k], tokens[1]) == 0){
						found = k;
						break;
					}
				}
				// replacing old command if found
				if(found != -1){
					free(alias_values[found]);
					alias_values[found] = command;
				}
				else{
					// resizing both arrays if needed
					if(alias_count >= alias_capacity){
						alias_capacity *=2;
						alias_names = realloc(alias_names, alias_capacity * sizeof(char*));
						alias_values = realloc(alias_values, alias_capacity * sizeof(char*));
						if(alias_names == NULL){
							perror("realloc");
							exit(1);
						}
						if(alias_values == NULL){
							perror("realloc");
							exit(1);
						}
					}
					// getting an own copy of name
					alias_names[alias_count] = strdup(tokens[1]);
					if(alias_names[alias_count] == NULL){
						perror("strdup");
						exit(1);
					}
					alias_values[alias_count] = command;
					alias_count++;
					}
				}
		}
		// executing unalias
		else if(strcmp(tokens[0], "unalias") == 0){
			if(tokens[1] == NULL){
				fprintf(stderr, "unalias needs an alias name\n");
			}
			else{
				// finding alias like before
				int found = -1;
				for(int k = 0; k < alias_count; k++){
					if(strcmp(alias_names[k], tokens[1]) == 0){
						found = k;
						break;
					}
				}
				// if not found not doing nything
				if(found != -1){
					// freeing(removing)
					free(alias_names[found]);
					free(alias_values[found]);
					// shifting all elements down
					for(int i = found; i < alias_count -1; i++){
						alias_names[i] = alias_names[i + 1];
						alias_values[i] = alias_values[i + 1];
					}
					alias_count --;
				}
			}
		}	
		// if none of the cases then we assume it is an external command
		else{
			// redirection targets
			char *out_file = NULL;
			int out_flags = 0;
			char *err_file = NULL;
			int err_flags = 0;
			int bad = 0;
			// checking number of pipe command and whether they have commands
			int num_cmds = 1;
			int has_cmd = 0;
			int v = 0;
			// looping to check for pipes
			// pipe syntax - command1 | command2
			// OUTPUT OF COMMAND 1 IS PASSED AS INPUT TO COMMAND2	
			while(tokens[v] != NULL && !bad){
				if(strcmp(tokens[v], "|") == 0){
					// nothing before the pipe
					if(!has_cmd){
						fprintf(stderr, "syntax error: missing command\n");
						bad = 1;
					}
					else{
						num_cmds ++;
						has_cmd = 0;
						v++;
					}
				}
				else if(is_redirect(tokens[v])){
					// if no file given or next token is a pipe
					if(tokens[v+1] == NULL || strcmp(tokens[v + 1], "|") == 0){
						fprintf(stderr, "syntax error: missing file after %s\n", tokens[v]);
						bad = 1;
					}
					else{
						// skip = and token[v]s operator
						v +=2;
					}
				}
				else{
					has_cmd = 1;
					v++;
				}
			}
			// last command may have nothing
			if(!bad && !has_cmd){
				fprintf(stderr, "syntax error: missing command\n");
				bad = 1;
			}
			// piping variables
			// process ids of children
			pid_t *pids = NULL;
			// how many children created
			int started = 0;
			// read end of prev pipe
			int prev_read = -1;
			// which command we are on
			int cmd = 0;
			
			// counter 
			int i = 0;
			if(!bad){
				// creating an array to store children for pipes
				pids = malloc(num_cmds * sizeof(pid_t));
				if(pids == NULL){
					perror("malloc");
					exit(1);
				}
				// flush buffer
				fflush(stdout);
			}
			while(!bad && cmd<num_cmds){
				// every commaand starts with no redirection
				out_file = NULL;
				out_flags = 0;
				err_file = NULL;
				err_flags = 0;
			// args hold only command and arguments
			char ** args = malloc((count + 1)* sizeof(char*));
			if(args == NULL){
				perror("malloc");
				exit(1);
			}
			// counters
			int a = 0;
			// pipe also added to checking
			while(tokens[i] != NULL && strcmp(tokens[i], "|") && !bad){
				if(strcmp(tokens[i], ">") == 0 || strcmp(tokens[i], ">>") == 0 || strcmp(tokens[i], "]") == 0 || strcmp(tokens[i], "]]") == 0){
					// if no file given
					if(tokens[i + 1] == NULL){
					fprintf(stderr, "syntax error: missing file after %s\n", tokens[i]);
					bad =1;
					}
					else{
						// > adds token[i] to token[i+1] file. WRONGLY -> opens file for writing
						// Create -> creates file, Trunc -> empties the file(overwrite), Append -> makes every write go to end
						if(strcmp(tokens[i], ">") == 0){
							out_file = tokens[i + 1];
							out_flags = O_WRONLY | O_CREAT | O_TRUNC;
						}
						// > adds token[i] to a new line after whats there to token[i + 1] file
						else if(strcmp(tokens[i], ">>") == 0){
							out_file = tokens[i + 1];
							out_flags = O_WRONLY | O_CREAT | O_APPEND;
							}
						// command nonsense ] file sends the error message to the file
						else if(strcmp(tokens[i], "]") == 0){
							err_file = tokens[i + 1];
							err_flags = O_WRONLY | O_CREAT | O_TRUNC;
							}
						// ]] appends the error to end of file
						else{
							err_file = tokens[i + 1];
							err_flags = O_WRONLY | O_CREAT | O_APPEND;
						}
						// skip operator and filename as already stored
						i += 2;
					}
				}
				else{
					// normal token goes to args
					args[a] = tokens[i];
					// counter of args increments
					a++;
					i++;
				}
			}
			// args last is NULL
			args[a] =NULL;
			// no command to run
			if(!bad && a ==0){
				fprintf(stderr, "syntax error: missing command\n");
				bad = 1;
			}
			// stopped at pipe. so there is a command
			int has_next = (tokens[i] != NULL);
			int pfd[2] = {-1, -1};
			if(!bad && has_next && pipe(pfd) == -1){
				perror("pipe");
				bad = 1;
			}
			// forking if syntax is fine
			// making sure it is an external command
			if(!bad){
				// storing 
				// flushing buffer
				fflush(stdout);
				pid_t rc = fork();
				if(rc < 0){
					perror("fork");
					// closing th epipe that was ade
					if(has_next){
						close(pfd[0]);
						close(pfd[1]);
					}
					bad = 1;
				}
				else if(rc == 0){
					if(prev_read != -1){
						// assigning the file STDIN_FILENO to prev read
						// dup2 just copies second file to first parameter
						if(dup2(prev_read, STDIN_FILENO) == -1){
							perror("dup2");
							_exit(1);
						}
						close(prev_read);
					}
					// output has to go to input of next command
					if(has_next){
						//storing stdoutfileno in pfd 1
						if(dup2(pfd[1], STDOUT_FILENO) == -1){
							perror("dup2");
							_exit(1);
						}
						close(pfd[0]);
						close(pfd[1]);
					}
					if(out_file != NULL){
						// opening out file if needed
						int fd = open(out_file, out_flags, 0644);
						if(fd == -1){
							perror(out_file);
							_exit(1);
						}
						if(dup2(fd, STDOUT_FILENO) == -1){
							perror("dup2");
							_exit(1);
						}
						close(fd);
					}
					// opening err_file
					if(err_file != NULL){
						int fd = open(err_file, err_flags, 0644);
                         if(fd == -1){
                             perror(err_file);
                             _exit(1);
                         }
                         if(dup2(fd, STDERR_FILENO) == -1){
                             perror("dup2");
                             _exit(1);
                         }
                         close(fd);
                     }
					// child process
					// giving child the parents environ
					environ = my_env;
					// execvp searches directories in PATH to find args[0] and shifts to it with rest of input
					execvp(args[0], args);
					// if it returns, then error
					// if gibberish then handled 
					perror(args[0]);
					_exit(1);
				}
				else{
					// parent saves the pid instead of waiting
						pids[started] = rc;
						started ++;
						// previous read end is not needed 
						if(prev_read != -1){
							close(prev_read);	
						}
						if(has_next){
							// write end must be closed
							close(pfd[1]);
							prev_read = pfd[0];
						}
						else{
							prev_read = -1;
						}
				}
			}
			// freeing args is enough as args[i] doesnt use malloc
			free(args);
			if(has_next){
				i++;
			}
			cmd ++;
		}
		// closing a read left over if loop stopped early
		if(prev_read != -1){
			close(prev_read);
		}
		// waiting for every child
		for(int k = 0; k < started; k ++){
			int status;
			waitpid(pids[k], &status, 0);
		}
		free(pids);
		}
		// freeing token so that they can be reused
		for(int i = 0; i < count; i++){
			free(tokens[i]);
		}
		// Exiting code if needed
		if(should_exit){
			break;
		}
	}
	// Freeing everything (Cleanup)
	for(int i = 0; i < e_count; i++){
		free(my_env[i]);
	}
	free(my_env);
	// freeing aliases
	for(int i =0; i < alias_count; i++){
		free(alias_names[i]);
		free(alias_values[i]);
	}
	free(alias_names);
	free(alias_values);
	free(tokens);
	free(line);	
	return 0;
}	
