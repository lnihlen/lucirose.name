.. post:: Jun 13, 2023
   :tags: hadron
   :author: Luci
   :excerpt: 2

.. highlight:: SuperCollider

Hadron Postmortem
=================
According to the git history on the `project repository <https://github.com/hadron-sclang/hadron>`_, I stopped hacking
on `Hadron <https://hadron-sclang.org>`_ in late October of 2022. I had been slowly losing momentum on the project for a
while.

I didn't make any formal announcement about taking a break, but now I've been pinged more than once on the
`SuperCollider Forum <https://scsynth.org>`_ about the state of Hadron development, I thought I'd make a more formal
announcement about the end of the project, and do a project postmortem analysis.

But First, Mortem
-----------------
I have officially suspended development on Hadron. I'll update the repository and website to indicate that. There was
some interest in the sclang `parser <https://github.com/hadron-sclang/sprklr>`_ that I wrote, so I'll make sure to keep
the lights on there, but please consider the rest of the projects in the Hadron organization defunct.

Why Start Hadron?
-----------------
My motivations for the Hadron project fall into a few themes:

* **Practical Compiler Experience**: I first thought that sclang was a near-ideal language choice for a beginning
  compiler project. Most compiler texts implement a "toy" language primarily concerned with the *exposition
  of language features.* Practical application of the language is left as an exercise to the reader. By contrast,
  people use the SuperCollider language to make music, and with a sizeable extant code base, evaluating the success of
  Hadron as a SuperCollider implementation seemed (again, on first impression) simple.

* **Possible Utility**: Writing another C compiler or a Python interpreter for fun might have taught me as much about
  compiler engineering as working on Hadron. But with Hadron, it seemed at least plausible that people would consider
  trying the project out. If the project was successful, I think there was a possibility that over time it could have
  become the *de facto* SuperCollider interpreter, ultimately replacing sclang. I'm not saying I expected that outcome,
  just that I found the *plausibility* of that outcome exciting. Making software gets a lot more fun when it has users.

* **Hubris**: In my defense, it is one of the `three virtues <https://thethreevirtues.com>`_.

Why Stop?
---------

The Tyranny Of Hyrum's Law
^^^^^^^^^^^^^^^^^^^^^^^^^^
From working on Hadron and talking with SuperCollider users, I started to feel that if I wanted to maintain the
plausibility of Hadron's eventual adoption, it needed to behave as similarly to sclang as possible. `Hyrum's Law
<https://www.hyrumslaw.com/>`_ strongly reinforces this notion. The SuperCollider Class Library has code that relies
on these subtleties, meaning (to me) the community sees them as canonical. I started to find myself in situations where
I had to reverse-engineer idiosyncratic or even buggy sclang behavior and then devise a plan to imitate it while
maintaining the solid engineering practices and principles necessary to Hadron.

The best example of this is the method return operator ``^``. I spent a few weekends of development time stepping
through sclang code to understand that sclang preserves the *callee* frame pointer as part of the lexical scope of
functions, meaning that the following code::

   Fizz {
       *buzz {
           var func = { |val| ^val; };
           "beginning".postln;
           func.value("middle").postln;
           "end".postln;
       }
   }

   (
   Fizz.buzz;  // prints "beginning", evaluates as "middle"
   )

will not unwind the stack after completing ``buzz``. Instead, the method return operator inside ``func`` will return
its value to the *callee* of the method defining ``func``, the interpreter root frame. I can construct examples that
attempt to return to callee frames that have already exited. I consider this a defect in SuperCollider's language
design, against the principle that writing an invalid program from valid program inputs should not be possible.

Rather than revise the language, I learned that James McCartney used this feature to implement support for
exceptions in SuperCollider Class Library code that is still largely unmodified since he wrote it in 2004. The dangerous
stack behavior has other unforeseen consequences. Users have long filed bugs about exception handling, for example
`#232 <https://github.com/supercollider/supercollider/issues/232>`_ predates SuperCollider's move to GitHub in 2012.

Sclang is a stack-based interpreter, while Hadron is register-based and follows the C ABI, so it can freely mix
SuperCollider JIT and AoT C++ code. What results in weird edge case behaviors in the interpreter will instead result in
segmentation faults in Hadron. I could not design a safe emulation of this unsafe SuperCollider behavior, nor did I have
any desire to try. Impasse.

SuperCollider Governance
^^^^^^^^^^^^^^^^^^^^^^^^
Languages evolve, or they die. It is difficult to predict how language features will interact with each other, so by
necessity, language design is a practice of constant iteration and refinement. The unforeseen consequences of the method
return operator implementation are a great example of this, and in my opinion, the right thing to do here is a breaking
change.

Breaking changes require the developer to submit an RFC. I don't have the patience and emotional fortitude to endure the
slings and arrows of the RFC process. I admire the folks who have the dedication to carry one all the way through, and
I'm glad the community has a democratic approach to resolving disputes. I think some folks in our community are
downright mean to each other in disagreement, and it's frankly scared me away from the entire process. Lastly, I think
a consensus-driven process will likely be too slow and divisive to reform the language effectively.

There is a lengthy discussion on the forum about what the next major iteration of the SuperCollider language might look
like, with opinions running the gamut from "the software doesn't need improvement" to "tear everything down and start
over." I think the problem here is that there is no decision-making authority. Other languages break this logjam by
empowering committees to revise the language standard. That could work for sclang too, but who would appoint the
committee? What authority would they have?

Class Library and Language Interdependence
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The sclang interpreter and its class library are inseparably entangled. Holding Hadron to the standard that it should
compile and execute the class library code the same as sclang significantly constrained Hadron's design, often resulting
in increased complexity.

Furthermore, Hadron's different design and implementation means I also need to rewrite all of the primitive library
code. This rewrite would be inevitable for any nontrivial change to sclang, too. The primitive code is rife with
assumptions about sclang internals and implementation details. And the primitive C++ code is statically linked into
the sclang binary (along with all of its dependencies, like Qt), meaning that a drop-in replacement to sclang must
provide the same dependencies or suitable substitutes.

When I first started working on Hadron, I didn't understand what a monumental task the primitive re-implementation would
be. It dwarfs the already daunting task of re-implementing the language core. The sclang interpreter defines over 700
unique primitives. The ``supercollider/lang/``` directory in the repository has around 50K lines of C++ code, of which
*over 40K* define primitives. The interpreter is less than a fifth of the source code in sclang.

Opinions about the class library are as varied as about the next iteration of the language. I think the library
needs an architect or committee. I'm less concerned with their decisions and more concerned with the
decision-making process. The library is in desperate need of stewardship and consistent macro-scale decision-making.
Because the class library and language are so tightly coupled, the only path forward is to evolve them both.

The Supernova Problem
^^^^^^^^^^^^^^^^^^^^^
Between the abuse and the staggering technical debt, it is clear that the role of the SuperCollider maintainers is as
thankless as it is vital. I have the utmost respect for the current maintainers. As I gathered feedback from them during
Hadron's development, I became increasingly determined that Hadron would never add to their burdens.

One of the concerns the maintainers brought up with me surrounds the history of Supernova, the alternative audio
synthesis server. It has some interesting parallels to the direction Hadron was trending in. Supernova is arguably a
generational step forward in modern software engineering tools and techniques compared to scsynth. A single person
contributed most of the code and has since left the project. As a result, the maintainers are responsible for
twice the code surface area for no increase in overall features, with many subtle differences in implementations.

The way out of this trap is to build a vibrant developer community working on Hadron. I tried several times to recruit
contributors to Hadron and had a few discussions with interested parties but never received a single contribution.
Compiler work is some of the most challenging software engineering work I've ever done, and the SuperCollider community
is rightly focused elsewhere. Nothing attracts volunteers like success, and I suspect that if I had reached a viable
alpha implementation of Hadron, I would have likely had an easier time drawing in new talent. Perhaps we will never
know.

Work and Personal Life Changes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
I found my work on Hadron fulfilling enough to convince me to transfer to an engineering role on a compiler team at
Google. I'm excited by the opportunity to work with so many knowledgeable and experienced colleagues, and I feel that
the position is probably the most rewarding one I've had while at Google. That said, I joined a team of career-long
experts as a relative beginner. So, any sense of hubris I might have had dried up pretty fast. I have my contributions
to make to the team, but I find the LLVM codebase to be more opaque and intimidating than any other project I've worked
on.

In November 2022 (shortly after my final Hadron commit), I traveled to San Jose to attend the 2022 LLVM Developer
Meeting. I learned a lot and met some fascinating people, and it also served to emphasize my feeling that I need to
accelerate my involvement with LLVM. After the meeting, I concluded that if I was going to spend time on a personal
compiler project, it might be best to focus on an LLVM-based one.

The Long Road Home
------------------
I worked on Hadron for two and a half years, thoroughly enjoying myself, and I learned a lot. Hadron inspired me
to transition my career toward compilers, which has also been great. So I'm delighted to have spent the time the way
I did.

If the SuperCollider community ever appoints a committee of folks to design the next iteration or revise the class
library, I'd be happy to help in whatever capacity was wanted. Or, if some volunteers want to help Hadron with the
political side of the RFC process, and I could find a few additional contributors to de-risk the maintainer burden
problem, I might consider reopening the project. But I think those are two big "ifs."
