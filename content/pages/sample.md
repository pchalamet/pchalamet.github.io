---
layout: post
title: unstructured task_group sample
---

```
hugo new posts/my-first-post.md
```

```
for(var i=0; i<10; ++i)
{
    Console.WriteLine(i);
}
```


Example using unstructured task_group (recursive fibonacci computation). It’s pretty useless but this shows at least dynamic addition of tasks into the task_group.

C++
{{< highlight cpp >}}
#include "stdafx.h"
#include <ppl.h>
 
using namespace Concurrency;
 
void task_fibo(task_group& tasks, combinable<unsigned int>& comb, unsigned int n)
{
    if( 1 >= n )
    {
        comb.local() += n;
    }
    else
    {
        tasks.run([&, n] { task_fibo(tasks, comb, n-1); });
        tasks.run([&, n] { task_fibo(tasks, comb, n-2); });
    }
}
 
void task_fibo()
{
    unsigned int n = 30;
 
    task_group tasks;
    combinable<unsigned int> comb;
    tasks.run_and_wait([&] { task_fibo(tasks, comb, n); });
    unsigned int fibo_n = comb.combine( std::plus<unsigned int>() );
    std::cout << "Fibo(" << n << ") = " << fibo_n << std::endl;
}
{{< /highlight >}}

C#
{{< highlight csharp >}}
for(var i=0; i<10; ++i)
{
    Console.WriteLine(i);
}
{{< /highlight >}}

F#
{{< highlight ml >}}

let private (|MatchZeroOrMore|_|) c =
    match c with
    | '*' -> Some c
    | _ -> None

let rec private matchRec (content : char list) (pattern : char list) =
    let matchZeroOrMore remainingPattern =
        match content with
        | [] -> matchRec content remainingPattern
        | _ :: t1 -> if matchRec content remainingPattern then true // match 0 time
                     else matchRec t1 pattern // try match one more time

    let matchChar firstPatternChar remainingPattern =
        match content with
        | firstContentChar :: remainingContent when firstContentChar = firstPatternChar -> matchRec remainingContent remainingPattern
        | _ -> false

    match pattern with
    | [] -> content = []
    | MatchZeroOrMore _ :: tail -> matchZeroOrMore tail
    | head :: tail -> matchChar head tail

let Match (content : string) (pattern : string) =
    matchRec (content.ToLowerInvariant() |> Seq.toList) (pattern.ToLowerInvariant() |> Seq.toList)
{{< /highlight >}}
