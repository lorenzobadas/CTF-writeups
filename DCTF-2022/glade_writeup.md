# Glade

CTF: DCTF 2022
Category: rev
Points: 400

## Description

---

> In this day and age, software writes software... but humans still write software that writes software.
> 

Solution has to be sent to a server that will give back the flag if it is correct

## Solution

---

### Launching the program

We are given a 64-bit ELF executable.

When we launch it:

```bash
renzo@UbuntuWSL:~$ ./glade
The outside world awaits.
```

And we are prompted for an input.

### Decompiled file

In the decompiled file we see that we can input up to 100 characters.

```c
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  char v4[104]; // [rsp+10h] [rbp-70h] BYREF
  unsigned __int64 v5; // [rsp+78h] [rbp-8h]

  v5 = __readfsqword(0x28u);
  puts("The outside world awaits.");
  __isoc99_scanf("%100s", v4);
  sub_113F(main, v4, sub_121B - sub_113F);
  return 0LL;
}
```

After taking the input `main` calls `sub_113F` that takes the address of `main` , the string we provided and the distance between the function `sub_121B` and `sub_113F`.

Actually since `sub_121B` comes right after `sub_113F` that difference is just the length of `sub_113F`, that is 220 bytes.

Before trying to understand what `sub_113F` does, we notice that the file has a lot of similar functions placed one after the other and the last one, `sub_315D3`, calls `system("cat flag.txt")`, so our goal is definitely to get to that function but there are no direct calls to it.

Let’s see what `sub_113F` does:

```c
__int64 __fastcall sub_113F(__int64 main_addr, char *current_char, __int64 length)
{
  char current_char_copy; // [rsp+2Bh] [rbp-15h]
  int v6; // [rsp+2Ch] [rbp-14h]
  int v7; // [rsp+30h] [rbp-10h]

  check1();

  v6 = 0;
  v7 = 0;
  switch ( *current_char )
  {
    case 'R':
      v7 = 1;
      break;
    case 'D':
      v6 = 1;
      break;
    case 'L':
      v7 = -1;
      break;
    case 'U':
      v6 = -1;
      break;
  }
  return ((sub_113F + 30 * length * v6 + length * v7))(sub_113F, current_char + 1, length);
}
```

First of all it calls a function that does a check of some sort, we’ll see that in a moment.

Then depending on the character that `current_char` points to it sets either the variable `v6` or `v7` to $\pm1$.

These variables control a call to a function that, similarly to the previous call, take the address of the caller, a pointer to our string (incremented by one so that it takes the next character), and the length of the function.

If we look at the next functions in memory, the exact same pattern is followed, every function is the same length and the only thing that changes is the check function at the beginning of every function.

Let’s take a look at the possibilities we have at the return statement to jump where we want.

```c
'R' -> call to sub_113F+length         // go 1 function forward
'L' -> call to sub_113F-length         // go 1 function backward
'D' -> call to sub_113F+(30*length)    // go 30 functions forward
'U' -> call to sub_113F-(30*length)    // go 30 functions backward
```

At this point we might speculate that *R* stands for *right, L* for *left, D* for *down* and *U* for *up*. So if, to move up, we go 30 functions backward that means a row of functions is 30 functions long. Also if we count this kind of functions in the program we notice they are exactly 900, that is $30\times30$. This way we can imagine the functions in a 30-by-30 matrix where in the top left-hand corner we have `sub_113F` and in the bottom right-hand corner we have our target function `sub_315D3`.

However if we try to get to the target function in the most straight forward manner the program exits. That is because of the check functions we mentioned earlier.

Let’s take a look to one of these check functions:

```c
__int64 __fastcall check_D(__int64 callee_f, __int64 caller_f, int length, int row_len)
{
  __int64 result; // rax

  result = callee_f - row_len * length;
  if ( caller_f != result )
  {
    if ( caller_f == length + callee_f )
      exit(-1);
    if ( caller_f == row_len * length + callee_f )
      exit(-1);
    if ( caller_f == callee_f - length )
      exit(-1);
    exit(-1);
  }
  return result;
}
```

This function exits whenever the previous function isn’t 30 functions backward. Basically the only way to get to a function that has this check is with a D move.

There are 16 of these check functions, one for every possible combination of allowed moves.

At this point we have a maze to solve.

Just as a note, the first function doesn’t actually check anything since there is no reason to exit without having made a single move, the reason it has nevertheless a call to a check function is just to keep the function 220 bytes long.

### Solving the maze

First of all we have to associate to each element of the matrix its conditions. From IDA we can get a list of every function that calls a certain check, and from there we build the matrix.

This is the matrix, where each element of it contains the moves allowed to get there.

```python
matrix = ['XXXX','LR  ','LUR ','R   ','LU  ','LUR ','R   ','U   ','U   ','LU  ','LUR ','LR  ','LUR ','LUR ','UR  ','U   ','U   ','LU  ','LR  ','UR  ','LU  ','UR  ','LU  ','R   ','U   ','LU  ','LR  ','R   ','LU  ','R   ',
'D   ','LU  ','DUR ','U   ','DU  ','DL  ','LUR ','DR  ','DLU ','DUR ','DLU ','R   ','DU  ','D   ','DLU ','DLUR','DUR ','DLU ','R   ','DU  ','DU  ','DU  ','DLU ','LUR ','DLUR','DLR ','R   ','LU  ','DLR ','UR  ',
'LU  ','DR  ','DLU ','DR  ','D   ','LU  ','DLUR','R   ','D   ','DU  ','D   ','L   ','DR  ','L   ','DUR ','D   ','D   ','D   ','L   ','DLR ','DR  ','DL  ','DUR ','DU  ','DL  ','LR  ','R   ','DU  ','LU  ','DUR ',
'DL  ','UR  ','DL  ','R   ','LU  ','DR  ','DU  ','U   ','U   ','DU  ','LU  ','R   ','U   ','L   ','DR  ','L   ','UR  ','U   ','L   ','UR  ','LU  ','LR  ','DUR ','DL  ','LUR ','R   ','LU  ','DR  ','DU  ','DU  ',
'LU  ','DLUR','LR  ','LR  ','DR  ','LU  ','DLR ','DLUR','DLUR','DLR ','DLUR','R   ','DL  ','LUR ','LUR ','R   ','DU  ','DLU ','R   ','DU  ','D   ','LU  ','DR  ','L   ','DLUR','R   ','D   ','LU  ','DR  ','DU  ',
'D   ','DLU ','LR  ','UR  ','LU  ','DR  ','U   ','DU  ','D   ','L   ','DUR ','LU  ','LUR ','DUR ','DU  ','LU  ','DR  ','DU  ','LU  ','DR  ','LU  ','DLR ','R   ','L   ','DLR ','UR  ','L   ','DUR ','L   ','DUR ',
'LU  ','DUR ','U   ','D   ','D   ','LU  ','DUR ','DLU ','R   ','L   ','DLR ','DUR ','D   ','DU  ','DL  ','DUR ','LU  ','DLUR','DLUR','LR  ','DR  ','L   ','LR  ','LR  ','LUR ','DLR ','LR  ','DUR ','L   ','DUR ',
'DU  ','DU  ','DLU ','UR  ','LU  ','DR  ','DL  ','DLUR','LUR ','LR  ','R   ','DU  ','L   ','DLUR','R   ','DLU ','DUR ','D   ','D   ','U   ','LU  ','R   ','U   ','U   ','DU  ','U   ','L   ','DLR ','UR  ','D   ',
'D   ','D   ','D   ','DLU ','DLR ','R   ','U   ','D   ','DLU ','UR  ','U   ','DL  ','UR  ','D   ','U   ','D   ','DU  ','U   ','L   ','DLR ','DLR ','UR  ','DU  ','DU  ','DLU ','DLR ','R   ','U   ','DL  ','R   ',
'LU  ','UR  ','L   ','DUR ','U   ','U   ','DL  ','LUR ','DR  ','DL  ','DR  ','U   ','DL  ','LUR ','DR  ','L   ','DUR ','DLU ','LR  ','UR  ','LU  ','DUR ','DU  ','DL  ','DR  ','LU  ','UR  ','DLU ','LR  ','UR  ',
'DU  ','DL  ','R   ','DLU ','DUR ','DU  ','L   ','DLR ','LR  ','R   ','U   ','DLU ','LR  ','DUR ','L   ','LUR ','DLR ','DR  ','L   ','DLUR','DR  ','DLU ','DUR ','U   ','U   ','DU  ','DLU ','DR  ','U   ','DU  ',
'DL  ','LR  ','UR  ','DU  ','DL  ','DLR ','LR  ','R   ','LU  ','R   ','DU  ','DLU ','R   ','DLU ','R   ','DLU ','LR  ','UR  ','U   ','DL  ','UR  ','DU  ','D   ','DL  ','DLR ','DUR ','D   ','U   ','DU  ','DU  ',
'U   ','U   ','DL  ','DLUR','LR  ','LR  ','LR  ','LR  ','DLR ','LR  ','DUR ','D   ','L   ','DLR ','R   ','DL  ','R   ','D   ','DU  ','U   ','DU  ','DLU ','LR  ','LUR ','R   ','DL  ','R   ','DLU ','DR  ','DU  ',
'DL  ','DLR ','UR  ','DU  ','LU  ','LUR ','LR  ','R   ','U   ','LU  ','DLR ','R   ','U   ','LU  ','R   ','U   ','LU  ','R   ','DLU ','DLR ','DUR ','DU  ','LU  ','DLR ','LUR ','UR  ','LU  ','DUR ','LU  ','DUR ',
'U   ','LU  ','DLUR','DLUR','DR  ','DLU ','LUR ','UR  ','DL  ','DLUR','R   ','LU  ','DLUR','DR  ','LU  ','DUR ','DLU ','UR  ','D   ','L   ','DUR ','D   ','D   ','L   ','DUR ','DU  ','DU  ','DL  ','DUR ','D   ',
'DL  ','DR  ','D   ','DLU ','R   ','D   ','DU  ','DL  ','R   ','DL  ','R   ','D   ','DLU ','R   ','D   ','DL  ','DUR ','DU  ','L   ','LR  ','DUR ','U   ','U   ','U   ','D   ','DU  ','DU  ','LU  ','DLUR','UR  ',
'L   ','LR  ','LR  ','DLUR','R   ','L   ','DR  ','LU  ','R   ','LU  ','UR  ','L   ','DUR ','LU  ','LR  ','LUR ','DUR ','DU  ','L   ','LR  ','DLR ','DLUR','DR  ','DL  ','UR  ','D   ','D   ','D   ','D   ','DU  ',
'LU  ','LR  ','LUR ','DLR ','LR  ','LUR ','LUR ','DLR ','LR  ','DR  ','D   ','LU  ','DLR ','DR  ','U   ','DU  ','DU  ','DLU ','R   ','LU  ','R   ','D   ','LU  ','R   ','DLU ','LUR ','UR  ','L   ','LR  ','DUR ',
'D   ','L   ','DLUR','LUR ','R   ','D   ','DL  ','UR  ','U   ','LU  ','LUR ','DUR ','U   ','L   ','DLUR','DR  ','D   ','DLU ','LUR ','DUR ','LU  ','LR  ','DR  ','L   ','DR  ','DU  ','D   ','L   ','LUR ','DR  ',
'U   ','L   ','DUR ','D   ','L   ','UR  ','LU  ','DLR ','DLR ','DR  ','D   ','D   ','DLU ','R   ','D   ','LU  ','LUR ','DUR ','D   ','D   ','DL  ','LUR ','LR  ','LR  ','LUR ','DLR ','R   ','U   ','DU  ','U   ',
'DU  ','U   ','D   ','L   ','LR  ','DLR ','DLUR','R   ','U   ','LU  ','UR  ','LU  ','DLR ','LR  ','LR  ','DUR ','DU  ','DLU ','LUR ','UR  ','LU  ','DR  ','U   ','L   ','DLUR','R   ','LU  ','DLR ','DLUR','DR  ',
'DL  ','DLR ','LUR ','LR  ','R   ','U   ','DL  ','R   ','DL  ','DR  ','DL  ','DLR ','R   ','L   ','LR  ','DUR ','D   ','D   ','D   ','DL  ','DLR ','LR  ','DLUR','R   ','DLU ','LR  ','DUR ','LU  ','DLUR','UR  ',
'LU  ','R   ','DU  ','L   ','LR  ','DLUR','LUR ','LR  ','R   ','L   ','LR  ','LUR ','LUR ','LUR ','R   ','DL  ','R   ','LU  ','LUR ','LR  ','LUR ','LUR ','DLUR','R   ','D   ','U   ','D   ','D   ','DU  ','D   ',
'DL  ','LUR ','DLR ','UR  ','U   ','DU  ','DL  ','R   ','LU  ','LR  ','LR  ','DR  ','DU  ','DL  ','LR  ','LR  ','LR  ','DUR ','DU  ','LU  ','DR  ','D   ','DLU ','UR  ','L   ','DLUR','LUR ','R   ','DU  ','U   ',
'L   ','DUR ','LU  ','DLUR','DLUR','DUR ','L   ','LR  ','DUR ','L   ','LUR ','LR  ','DLR ','R   ','L   ','LUR ','R   ','DU  ','D   ','DLU ','R   ','U   ','D   ','DL  ','LUR ','DR  ','D   ','LU  ','DLUR','DR  ',
'L   ','DUR ','DU  ','D   ','DU  ','DLU ','UR  ','LU  ','DR  ','U   ','DL  ','R   ','LU  ','UR  ','L   ','DLUR','LUR ','DR  ','L   ','DLR ','LUR ','DR  ','LU  ','UR  ','D   ','L   ','LUR ','DR  ','DL  ','UR  ',
'U   ','D   ','DU  ','L   ','DR  ','D   ','DL  ','DLR ','LR  ','DUR ','U   ','U   ','D   ','DL  ','LR  ','DUR ','D   ','LU  ','R   ','LU  ','DLR ','R   ','DU  ','DLU ','LR  ','LR  ','DLR ','R   ','LU  ','DUR ',
'DLU ','LR  ','DR  ','L   ','LUR ','LUR ','UR  ','U   ','U   ','DL  ','DLR ','DLUR','LR  ','LR  ','R   ','D   ','LU  ','DUR ','LU  ','DUR ','LU  ','LR  ','DUR ','DL  ','LR  ','LUR ','LR  ','R   ','DU  ','DU  ',
'DLU ','R   ','LU  ','LR  ','DUR ','D   ','DLU ','DLR ','DUR ','U   ','L   ','DUR ','U   ','U   ','L   ','UR  ','DU  ','DU  ','D   ','DU  ','D   ','LU  ','DLR ','UR  ','U   ','DU  ','U   ','U   ','DU  ','D   ',
'DL  ','R   ','D   ','L   ','DR  ','L   ','DR  ','L   ','DLR ','DLR ','LR  ','DLR ','DLR ','DLR ','LR  ','DLR ','DR  ','D   ','L   ','DR  ','L   ','DR  ','L   ','DLR ','DR  ','D   ','DL  ','DLR ','DLR ','R   ']
```

We approached this maze with a naïve solution that, starting from `matrix[899]` (our target function) just checks every allowed move until it gets to `matrix[0]`, and then we started optimizing the algorithm from there. Thankfully we managed to get the solution only with 3 checks.

- Make sure we are inside the bounds of `matrix`
- Make sure we have enough moves left to get to `matrix[0]` since we only have 100 moves
- Make sure we are not stuck in a loop of moves that would lengthen the times unnecessarily

If these check are not satisfied we abort that specific path e go back to the previous move to find another way.

This is the final script to find a path from `matrix[899]` to `matrix[0]`:

```python
def valid_pos(pos):
    if pos in range(900):
        return True
    return False

def find_way(matrix, pos, old, moves):
    if pos == 0:
        print(''.join(moves[::-1])) # reverse moves since we are starting from the end to solve the maze
        exit()
    
    if not valid_pos(pos):
        return False
    
    if pos in old[-2:]:             # check if we are stuck in a loop of moves
        return False

    distance = pos%30 + pos//30
    if 100 - len(moves) < distance: # check if we have enough moves left to get to matrix[0]
        return False

    dirs = matrix[pos].strip()
    
    for dir in dirs:
        if dir == 'D':
            find_way(matrix, pos-30, old[-2:] + [pos], moves + ['D'])
        elif dir == 'U':
            find_way(matrix, pos+30, old[-2:] + [pos], moves + ['U'])
        elif dir == 'R':
            find_way(matrix, pos-1, old[-2:] + [pos], moves + ['R'])
        elif dir == 'L':
            find_way(matrix, pos+1, old[-2:] + [pos], moves + ['L'])

    return False

find_way(matrix, 899, [], [])
```

Running this script we get the string `RRDLDLDRDRRRURURDDRDDDLULDLDLDDDDDDDDDRRRDRDRRURRURRURRRUURDDDDDDRRDRURURRRDDRRURRDDDDDRDLDDDR` that is indeed a valid path to get from `matrix[0]` to `matrix[899]`.

Sending this string to the server will give us the flag.
