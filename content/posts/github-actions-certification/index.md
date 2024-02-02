---
title: "Taking the Github Actions Certification Exam"
date: 2024-02-02T22:43:00+01:00

description: |
  My experience with the Github Actions Certification.
summary: |
  My experience with the Github Actions Certification.

tags:
  - github
  - cicd
  - certification
  - devops
---

GitHub [announced on the 8th of January 2024][announcement] that they are now also in the certification business, offering four new certifications to the general public.
The four certifications offered are:

- [GitHub Foundations][foundations]
- [GitHub Actions][actions]
- [GitHub Advanced Security][security]
- [GitHub Administration][administration]

Some more information about the certifications can also be found on [this GitHub Resources page][resources] dedicated to the certifications. Information is spread all over the place, some of it on GitHub, and some of it, e.g. the above links to the certifications, on Microsoft Learn.

While I am not a big fan of certifications in general as they are often a poor indicator of actual practical knowledge and experience, I do recognize that many recruiters and companies use them as a filter, and you never know how much they might help you in the future, especially if you're still early in your career.

As I am working quite a lot with GitHub Actions in my day-to-day job, that seemed like the best fit for me.

## GitHub Actions Certification - What to expect

The [Certification program FAQ][faq] states that the certification is a 120-minute, 60-question plus 10-15 pre-test questions, multiple choice exam. The exam can be taken both in-person at a PSI testing center or online as a proctored exam. The exam costs $200, currently discounted to $99. The certification expires after three years.

The passing score is not explicitly stated, as it's claimed to vary based on the specific exam taken. It's reasonable to assume that it's somewhere around 70%.

## Studying for the exam

As I am working with GitHub Actions on a daily basis, I did not need to spend much time studying for the exam. As the certifications are also a quite new addition, even though they have been in closed beta for a while, there is not much information third-party information available on the internet.

The following resources were helpful for me, even though I did not spend much time with them:

- [Microsoft Learn curriculum on GitHub Actions][actions] - I've given this a quick read as it's the officially recommended resource. It gives a good overview and has some interactive exercises. Definitely worthwhile, especially if you're new to GitHub Actions.
- The [study guide][study-guide] is linked from various places on the GitHub website, living in the [`LadyKerr/github-certification-guide`][study-guide] repository. It gives a short syllabus-style overview of the topics covered in the exam.
- The [official GitHub Actions documentation][gh-actions-docs] is probably the best resource for learning the theoretical aspects of GitHub Actions. It covers all the topics in the exam, but might be a bit dry to read through 😉.
- [GHCertified.com][ghcertified] by [Aleksander Fidelius][aleksander-fidelius] is a nifty practice exam for the GitHub Actions certification based on crowdsourced questions. Additions can be made via pull requests on [the GitHub repository][ghcertified-repo].

## My experience with the exam

As the next local testing center is at least half an hour away, I opted for the online proctored exam option. As is usual for these kinds of exams, special exam software needs to be installed on the computer and the exam is proctored via webcam and microphone. Nothing out of the ordinary here, but might be a bit of a hassle if you're not used to it. All in all a smooth experience.

I took the exam on an early Friday afternoon and finished it in **about 27 minutes** out of the allotted 120 minutes. The exam itself was quite straightforward. The questions were not too tricky, and I did not feel like I was being tricked into answering something wrong; most of the time.

{{<
  figure
  src="certification.webp"
  alt="GitHub Actions Certification"
  caption="My final score was 55 out of 60 questions. During the exam, I did not expect to do that well."
  class="rounded-lg"
>}}

### Was it worth it?

Probably, who knows? Hard to say.

While I did not get much out of the exam itself, I did not spend much time studying for it either. Your mileage may vary. If you're new to GitHub Actions, it might be a good way to get a good understanding of the topic and its ecosystem. And you get a Credly badge to brag about on LinkedIn, so that's something, I guess. :medal_military:

[announcement]: https://github.blog/2024-01-08-github-certifications-are-generally-available/ "GitHub Certifications are generally available"
[foundations]: https://learn.microsoft.com/en-us/collections/o1njfe825p602p "GitHub Foundations Certification"
[actions]: https://learn.microsoft.com/en-us/collections/n5p4a5z7keznp5 "GitHub Actions Certification"
[security]: https://learn.microsoft.com/en-us/collections/rqymc6yw8q5rey "GitHub Advanced Security Certification"
[administration]: https://learn.microsoft.com/en-us/collections/mom7u1gzjdxw03 "GitHub Administration Certification"
[resources]: https://resources.github.com/learn/certifications/ "GitHub Certification Overview"
[faq]: https://examregistration.github.com/faq "GitHub Certification FAQ"
[study-guide]: https://github.com/LadyKerr/github-certification-guide/blob/main/study-guides/gh-actions.md "GitHub Actions Study Guide"
[gh-actions-docs]: https://docs.github.com/en/actions "GitHub Actions Documentation"
[ghcertified]: https://ghcertified.com/ "GHCertified"
[aleksander-fidelius]: https://github.com/FidelusAleksander "Alexander Fidelius on GitHub"
[ghcertified-repo]: https://github.com/FidelusAleksander/ghcertified "GHCertified GitHub Repository"
