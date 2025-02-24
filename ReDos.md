# Security Briefs - Regular Expression Denial of Service Attacks and Defenses

By [Bryan Sullivan](https://learn.microsoft.com/en-us/archive/msdn-magazine/2010/may/\archive\msdn-magazine\authors\Bryan_Sullivan) | May 2010

![Bryan Sullivan](./images/ReDos/ff646973.sullivan_headshot(en-us).png)In the November 2009 issue, I wrote an article titled “XML Denial of Service Attacks and Defenses” ([msdn.microsoft.com/magazine/ee335713](https://msdn.microsoft.com/en-us/ee335713.aspx)), in which I described some particularly effective denial of service  (DoS) attack techniques against XML parsers. I received a lot of e-mail  about this article from readers wanting to know more, which really  encourages me that people understand how serious DoS attacks can be.

I believe that in the next four to five years, as privilege  escalation attacks become more difficult to execute due to increased  adoption of memory protections such as Data Execution Prevention (DEP),  Address Space Layout Randomization (ASLR), and isolation and privilege  reduction techniques, attackers will shift their focus to DoS blackmail  attacks. Developers can continue to protect their applications by  staying ahead of the attack trend curve and addressing potential future  DoS vectors today.

One of those potential future DoS vectors is the regular expression  DoS. At the Open Web Application Security Project (OWASP) Israel  Conference 2009, Checkmarx Chief Architect Alex Roichman and Senior  Programmer Adar Weidman presented some excellent research on the topic  of regular expression DoS, or “ReDoS.” Their research revealed that a  poorly written regular expression can be exploited so that a relatively  short attack string (fewer than 50 characters) can take hours or more to evaluate. In the worst-case scenario, the processing time is actually  exponential to the number of characters in the input string, meaning  that adding a single character to the string doubles the processing  time.

In this article, I will describe what makes a regex vulnerable to  these attacks. I will also present code for a Regex Fuzzer, a test  utility designed to identify vulnerable regexes by evaluating them  against thousands of random inputs and flagging whether any of the  inputs take an unacceptably long time to complete processing.

(Note: For this article, I assume you are familiar with the syntax of regular expressions. If this is not the case, you might want to brush  up by reading the article “.NET Framework Regular Expressions” at [msdn.microsoft.com/library/hs600312](https://msdn.microsoft.com/en-us/library/hs600312.aspx), or for a deeper dive, read Jeffrey Friedl’s excellent reference book,  “Mastering Regular Expressions 3rd Edition” [O’Reilly, 2006].)



## Backtracking: The Root of the Problem

There are essentially two different types of regular expression  engines: Deterministic Finite Automaton (DFA) engines and  Nondeterministic Finite Automaton (NFA) engines. A complete analysis of  the differences between these two engine types is beyond the scope of  this article; we only need to focus on two facts:

1. NFA engines are backtracking engines. Unlike DFAs, which evaluate  each character in an input string at most one time, NFA engines can  evaluate each character in an input string multiple times. (I’ll later  demonstrate how this backtracking evaluation algorithm works.) The  backtracking approach has benefits, in that these engines can process  more-complex  regular expressions, such as those containing  backreferences or capturing parentheses. It also has drawbacks, in that  their processing time can far exceed that of DFAs.
2. The Microsoft .NET Framework System.Text.RegularExpression classes use NFA engines.

One important side effect of backtracking is that while the regex  engine can fairly quickly confirm a positive match (that is, an input  string does match a given regex), confirming a negative match (the input string does not match the regex) can take quite a bit longer. In fact,  the engine must confirm that none of the possible “paths” through the  input string match the regex, which means that all paths have to be  tested.

With a simple non-grouping regular expression, the time spent to  confirm negative matches is not a huge problem. For example, assume that the regular expression to be matched against is:

```
^\d+$
```

This is a fairly simple regex that matches if the entire input string is made up of only numeric characters. The ^ and $ characters represent the beginning and end of the string respectively, the expression \d  represents a numeric character, and + indicates that one or more  characters will match. Let’s test this expression using 123456X as an  input string.

This input string is obviously not a match, because X is not a  numeric character. But how many paths would the sample regex have to  evaluate to come to this conclusion? It would start its evaluation at  the beginning of the string and see that the character 1 is a valid  numeric character and matches the regex. It would then move on to the  character 2, which also would match. So the regex has matched the string 12 at this point. Next it would try 3 (and match 123), and so on until  it got to X, which would not match.

However, because our engine is a backtracking NFA engine, it does not give up at this point. Instead, it backs up from its current match  (123456) to its last known good match (12345) and tries again from  there. Because the next character after 5 is not the end of the string,  the regex is not a match, and it backs up to its previous last known  good match (1234) and tries again. This proceeds all the way until the  engine gets back to its first match (1) and finds that the character  after 1 is not the end of the string. At this point the regex gives up;  no match has been found.

All in all, the engine evaluated six paths: 123456, 12345, 1234, 123, 12 and 1. If the input string had been one character longer, the engine would have evaluated one more path. So this regular expression is a  linear algorithm against the length of the string and is not at risk of  causing a DoS. A System.Text.RegularExpressions.Regex object using ^\d+$ for its pattern is fast enough to tear through even enormous input  strings (more than 10,000 characters) virtually instantly.

Now let’s change the regular expression to group on the numeric characters:

```
^(\d+)$
```

This does not substantially change the outcome of the evaluations; it simply lets the developer access any match as a captured group. (This  technique can be useful in more complicated regular expressions where  you might want to apply repetition operators, but in this particular  case it has no value.) Adding grouping parentheses in this case does not substantially change the expression’s execution speed, either. Testing  the pattern against the input 123456X still causes the engine to  evaluate just six different paths. However, the situation is  dramatically different if we make one more tiny change to the regex:

```
^(\d+)+$
```

The extra + character after the group expression (\d+) tells the  regex engine to match any number of captured groups. The engine proceeds as before, getting to 123456 before backtracking to 12345.

Here is where things get “interesting” (as in horribly dangerous).  Instead of just checking that the next character after 5 is not the end  of the string, the engine treats the next character, 6, as a new capture group and starts rechecking from there. Once that route fails, it backs up to 1234 and then tries 56 as a separate capture group, then 5 and 6  each as separate capture groups. The end result is that the engine  actually ends up evaluating 32 different paths.

If we now add just one more numeric character to the evaluation  string, the engine will have to evaluate 64 paths—twice as many—to  determine that it’s not a match. This is an exponential increase in the  amount of work being performed by the regex engine. An attacker could  provide a relatively short input string—30 characters or so—and force  the engine to process hundreds of millions of paths, tying it up for  hours or days.



## Airing Your Dirty Laundry

It’s bad enough when an application has DoS-able exponential regexes  tucked away in server-side code. It’s even worse when an application  advertises its vulnerabilities in client-side code. Many of the ASP.NET  validator controls derived from System.Web.UI.WebControls.BaseValidator, including RegularExpressionValidator, will automatically execute the  same validation logic on the client in JavaScript as they do on the  server in .NET code.

Most of the time, this is a good thing. It’s good to save the user  the round-trip time of a form submission to the server just to have it  rejected by a validator because the user mistyped an input field. It’s  good to save the server the processing time, too. However, if the  application is using a bad regex in its server code, that bad regex is  also going to be used in its client code, and now it will be extremely  easy for an attacker to find that regex and develop an attack string for it.

For example, say I create a new Web form and add a TextBox and a  RegularExpressionValidator to that form. I set the validator’s  ControlToValidate property to the name of the text box and set its  ValidationExpression to one of the bad regexes I’ve discussed:

​		

```
this.RegularExpressionValidator1.ControlToValidate = "TextBox1";

this.RegularExpressionValidator1.ValidationExpression = @"^(\d+)+$";
```

If I now open this page in a browser and view its source, I see the following JavaScript code close to the bottom of the page:

​		

```
<scripttype="text/javascript">

//< ![CDATA[

var RegularExpressionValidator1 = document.all ? 

  document.all["RegularExpressionValidator1"] : 

  document.getElementById("RegularExpressionValidator1");

RegularExpressionValidator1.controltovalidate = "TextBox1";

RegularExpressionValidator1.validationexpression = "^(\\d+)+$";

//]] >

</script>
```

There it is, for the whole world to see: the exponential regex in plain sight on the last line of the script block.



## More Problem Patterns

Of course, ^(\d+)+$ is not the only bad regular expression in the  world. Basically, any regular expression containing a grouping  expression with repetition that is itself repeated is going to be  vulnerable. This includes regexes such as:

```
^(\d+)*$`
 `^(\d*)*$`
 `^(\d+|\s+)*$
```

In addition, any group containing alternation where the alternate subexpressions overlap one another is also vulnerable:

```
^(\d|\d\d)+$`
 `^(\d|\d?)+$
```

If you saw an expression like the previous sample in your code now,  you’d probably be able to identify it as vulnerable just from looking at it. But you might miss a vulnerability in a longer, more complicated  (and more realistic) expression:

```
^([0-9a-zA-Z]([-.\w]*[0-9a-zA-Z])*@(([0-9a-zA-Z])+([-\w]*[0-9a-zA-Z])*\.)+[a-zA-Z]{2,9})$
```

This is a regular expression found on the Regular Expression Library  Web site (regexlib.com) that is intended to be used to validate an  e-mail address. However, it’s also vulnerable to attack. You might find  this vulnerability through manual code inspection, or you might not. A  better technique to find these problems is required, and that’s what I’m going to discuss next.



## Finding Bad Regexes in Your Code

Ideally, there would be a way to find exponential regexes in your  code at compile time and warn you about them. Presumably, in order to  parse a regex string and analyze it for potential weaknesses, you would  need yet another regex. At this point, I feel like a regex addict: “I  don’t need help, I just need more regexes!” Sadly, my regex skills are  not up to the task of writing a regex to analyze regexes. If you believe you have working code for this, send it to me and I’ll be happy to give you credit in next month’s Security Briefs column. In the meantime,  because I don’t have a way to detect bad regexes at compile time, I’ll  do the next best thing: I’ll write a regex fuzzer.

Fuzzing is the process of supplying random, malformed data to an  application’s inputs to try to make it fail. The more fuzzing iterations you run, the better the chance you’ll find a bug, so it’s common for  teams to run thousands or millions of iterations per input. Microsoft  security teams have found this to be an incredibly effective way to find bugs, so the Security Development Lifecycle team has made fuzzing a  requirement for all product and service teams.

For my fuzzer, I want to fuzz random input strings to my regular  expression. I’ll start by defining a const string for my regex, a  testInputValue method that checks the regex and a runTest method that  will collect random input strings to feed to testInputValue.

​		

```
const string regexToTest = @"^(\d+)+$";



static void testInputValue(string inputToTest)

{

  System.Text.RegularExpressions.Regex.Match(inputToTest, regexToTest);

}



void runTest()

{

  string[] inputsToTest = {};



  foreach (string inputToTest in inputsToTest)

  testInputValue(inputToTest);

}
```

Note that there’s no code yet to generate the fuzzed input values;  I’ll get to that shortly. Also note that the code doesn’t bother to  check the return value from Regex.Match. This is because I don’t  actually care whether the input matches the pattern or not. All I care  about in this situation is whether the regex engine takes too long to  decide whether the input matches.

Normally fuzzers are used to try to find exploitable privilege  elevation vulnerabilities, but again, in this case, I’m only interested  in finding DoS vulnerabilities. I can’t simply feed my test application  data to see if it crashes; I have to be able to detect whether it’s  locked up. Although it may not be the most scientific method, I can  accomplish this effectively by running each regex test sequentially on a separate worker thread and setting a timeout value for that thread’s  completion. If the thread does not complete its processing within a  reasonable amount of time, say five seconds to test a single input, we  assume that the regular expression has been DoS’d. I’ll add a  ManualResetEvent and modify the testInputValue and runTest methods  accordingly, as shown in **Figure 1**.

Figure 1 Testing Using Separate Worker Threads

​		

```
static ManualResetEvent threadComplete = new ManualResetEvent(false);



static void testInputValue(object inputToTest)

{

  System.Text.RegularExpressions.Regex.Match((string)inputToTest, 

    regexToTest);

  threadComplete.Set();

}



void runTest()

{

  string[] inputsToTest = {};



  foreach (string inputToTest in inputsToTest)

  {

    Thread thread = new Thread(testInputValue);

    thread.Start(inputToTest);



    if (!threadComplete.WaitOne(5000))

    {

      Console.WriteLine("Regex exceeded time limit for input " + 

        inputToTest);

      return;

    }



    threadComplete.Reset();

  }



  Console.WriteLine("All tests succeeded within the time limit.");

}
```

Now it’s time to generate the input values. This is actually more  difficult than it sounds. If I just generate completely random data,  it’s unlikely any of it would match enough of the regex to reveal a  vulnerability. For example, if I test the regex ^(\d+)+$ with the input  XdO(*iLy@Lm4p$, the regex will instantly not match and the problem will  remain hidden. I need to generate input that’s fairly close to what the  application expects for the test to be useful, and for that I need a way to generate random data that matches a given regex.



## Data Generation Plans to the Rescue

Fortunately, there is a feature in Visual Studio Database Projects  that can do just that: the data generation plan. If you’re using Visual  Studio Team Suite, you also have access to this feature. Data generation plans are used to quickly fill databases with test data. They can fill  tables with random strings, or numeric values or (luckily for us)  strings matching specified regular expressions.

You first need to create a table in a SQL Server 2005 or 2008  database into which you can generate test data. Once that’s done, come  back into Visual Studio and create a new SQL Server Database project.  Edit the database project properties to provide it with a connection  string to your database. Once you’ve entered a connection string and  tested it to make sure it works, return to the Solution Explorer and add a new Data Generation Plan item to the project. At this point, you  should see something like **Figure 2**.

![Figure 2 A Data Generation Plan Item in Visual Studio](./images/ReDos/ff646973.sullivan_figure2(en-us,msdn.10).png)
 Figure 2 **A Data Generation Plan Item in Visual Studio**

Now choose the table and column you want to fill with fuzzer input  data. In the table section, set the number of test values to be  generated (the Rows to Insert column). I wrote earlier that fuzzers  generally test hundreds of thousands or millions of iterations to try to find problems. Though I usually approve of this level of rigor, it’s  overkill for the purposes of this regex fuzzer. If a regex is going to  lock up, it’s going to do so within a couple hundred test iterations. I  suggest setting the Rows to Insert value to 200, but if you want to test more, please feel free.

In the column section, now set the Generator to Regular Expression  and enter the regex pattern value you want to test as the value of the  Expression property in the column’s Properties tab. It’s important to  note that the Expression property doesn’t support every legal regular  expression character. You can’t enter the beginning- and end-of-line  anchors ^ and $ (or more accurately, you can enter them, but the  generator will generate a literal ^ or $ character in the test input).  Just leave these characters out. You’ll find a full list of operators  supported by the Regular Expression Generator at [msdn.microsoft.com/library/aa833197(VS.80)](https://msdn.microsoft.com/en-us/library/aa833197(vs.80)).

A bigger problem is that the Expression property also doesn’t support common shorthand notations such as \d for numeric digits or \w for word characters. If your regex uses these, you’ll have to replace them with  their character class equivalents: [0-9] instead of \d, [a-zA-Z0-9_]  instead of \w and so on. If you need to replace \s (whitespace  character), you can enter a literal space in its place.

Your last task in the database project is to actually fill the  database with the test data according to your specifications. Do this by choosing the Data | DataGenerator | Generate Data menu item, or just  press F5.



## Adding the Attack

Back in the fuzzer code, modify the runTest method so it pulls the  generated test data from the database. You might think you’re done after this, but in fact there’s one more important change to make. If you run the fuzzer now, even against a known bad regex such as ^(\d+)+$, it  will fail to find any problems and report that all tests succeeded. This is because all the test data you’ve generated is a valid match for your regex.

Remember earlier I stated that NFA regex engines can fairly quickly  confirm a positive match and that problems really only happen on  negative matches. Furthermore, because of NFAs’ backtracking nature,  problems only occur when there are a large number of matching characters at the start of the input and the bad character appears at the end. If a bad character appeared at the front of the input string, the test would finish instantly.

The final change to make to the fuzzer code is to append bad  characters onto the ends of the test inputs. Make a string array  containing numeric, alphabetic, punctuation and whitespace characters:

 

​		

```
string[] attackChars = { "0", "1", "9", "X", "x", "+", 

"-", "@", "!", "(", ")", "[", "]", "\\", "/", 

"?", "<", ">", ".", ",", ":", ";", " ", "" };
```

Now modify the code so each input string retrieved from the database  is tested with each of these attack characters appended to it. So the  first input string would be tested with a 0 character appended to it,  then with a 1 character appended to it and so on. Once that input string has been tested with each of the attack characters, move to the next  input string and test it with each of the attack characters.

​		

```
foreach (string inputToTest in inputsToTest)

{

  foreach (string attackChar in attackChars)

  {

    Threadthread = new Thread(testInputValue);

    thread.Start(inputToTest + attackChar);

...
```



## Now You Have Two Problems

There is a famous quote by ex-Netscape engineer Jamie Zawinski concerning regular expressions:

*“Some people, when confronted with a problem, think, ‘I know, I’ll use regular expressions.’ Now they have two problems.”*

While I am nowhere near as cynical about regexes as Mr. Zawinski, I  will admit that it can be quite challenging just to write a correct  regex, much less a correct regex that is secure against DoS attacks. I  encourage you to examine all of your regexes for exponential complexity, and to use fuzzing techniques to verify your findings.      

------

**Bryan Sullivan** *is a security program manager for the Microsoft Security Development Lifecycle team, where he specializes in Web application and .NET security issues. He is the author of “Ajax  Security” (Addison-Wesley, 2007).*

*Thanks to the following technical experts for reviewing this  article: Barclay Hill, Michael Howard, Ron Petrusha and Justin van  Patten*