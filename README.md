# BalsnCTF 2022

This repository includes files and writeups for challenges I authored in BalsnCTF 2022. The challenges are sorted by intended difficulty from easy to hard.

The solution scripts can be found in each challenge's folder. For certain challenges, the output can be generated by running `python3 chall.py`.

## Yet another RSA with hint

This one is pretty straightforward. The digit sum in base `b` gives you the information of `p mod (b - 1)`, just like how we sum all digits in base 10 to get the remainder mod 9. With help of CRT, we can get the remainder of `p` modulo `lcm(1, 2, ...)`. The little twist is that this is not enough to get an unique `p`, but it reveals more than half of the bits, so coppersmith method is sufficient to recover `p`.

## LFSR

Define `a` to be the state sequence and `o` to be the output sequence.

The first thing that looks sus is that all elements in `_p` are a multiple of 8. Since `a[0], a[8], a[16], ...` forms a state sequence of a LFSR with potentially different feedback polynomial/tap (some intuition: the fibonacci sequecne has linear relation `f[n+2] = f[n+1] + f[n]`, and the sequence `f[0], f[2], f[4], ...` satisfy another linear relation `f[n+4] = 3 * f[n+2] - f[n]`), we can "reduce" the problems to 8 LFSR with the same feedback polynomials, and the output is the evaluation of that weird polynomial with input `a[0], a[1], a[2], a[4], a[8], a[15]`.

(Excercise: Prove that since 8 is a power of 2, the feedback polynomial for the seperated LFSR is actually the same as the original one, which is `x^128 + x^7 + x^2 + x + 1`)

Now we split a single problem into 8, what do we get? The only benefit is that the output depends on a smaller interval of states, which allows the following attack idea.

The idea is essentially a DFS with branch-cutting. We bruteforce the first 15 state bits `a[0], a[1], ..., a[14]` and try to determine the value of `a[15]` based on the first output bits, then determine `a[16]` by the second output bits, and so on. Given `a[0], a[1], ..., a[14]` and the first output `o[0]`, there are three scenarios:

* Both 0, 1 are possible for `a[15]`, then we make two branches and go on.
* Value of `a[15]` is unique, then just keep going.
* None of 0, 1 work, then we cut this branch.

We can see that case 1 increase the number of paths we need to explore by 1, while case 3 eliminates a path. If we treat all `a[i]` as independent random bits, the probability of case 1 and case 3 are the same, meaning that on expectation, the number of paths to explore is a constant. Therefore, we can find all possible initial states (128 bits), and simulate the LFSR to see if the outputs match, recovering the initial states to decrypt the flag.

(Remark 1: a more detailed analysis on the number of paths can bound the maximum instead of expectation – in each level, the number of paths increase by O(sqrt(n)) with high probability by normal distribution approximation)

(Remark 2: I didn't realize that case 1 and case 3 kinda cancelling out, so the polynomial was unnecessarily designed to avoid case 1)

## VSS

It is quite clear that the commitment scheme has a huge issue - `p` is not a strong prime, and the base `g` is not always a quadratic residue. So each commitment leaks a partial information of the number. Specifically, if `o` is the smooth part of `p-1` (meaning that `o | p - 1` and `o` is smooth), then we can leak the value of `val % o`.

With enough commitment, we can get a bunch of equations of the form `(a + b * s1 + c * s2) % p = r + o * x`, where `p, r, o, a, b, c` are known, and `x < p / o`. Since the `p` is different in each commitment, we can use CRT to merge the equations to a single one `(A + B * s1 + C * s2) % P = R + o1 * x1 + o2 * x2 + ...`. Compare to the modulus `P`, `xi`, `s1`, and `s2` are small, so we can use LLL to recover the two secret values.

The CRT part might be confusing so here's an example with `a = 0, b = 1, c = 0`. Assume that the equations are
`s1 %  7 = 1 + 2 * x1`
`s1 % 11 = 2 + 3 * x2`
`s1 % 13 = 3 + 5 * x3`

Then we can treat it as 
`s1 %  7 = 1 + 2 * x1 + 0 * x2 + 0 * x3`
`s1 % 11 = 2 + 0 * x1 + 3 * x2 + 0 * x3`
`s1 % 13 = 3 + 0 * x1 + 0 * x2 + 5 * x3`

and combined them to

`s1 = 211 + 429 * x1 + 91 * x2 + 616 * x3 (mod 1001)`

In the official solution, we always choose `p` such that the smooth part is at least 40-bit long, and with 30 of them, it's enough to recover the secrets.

(Remark 1: I didn't know that random DLP is not too hard with 512 bits, nor do I know that leaking 70 bits from a single commitment is do-able.)

(Remark 2: The original challenge idea is to only leak the LSB by setting strong `p`, but non-quadratic-residue `g`, however, as a newbie in LLL, I'm not confident that it is solvable with 128-bit secrets because of the huge dimension of the matrix, and 64-bit secret is probably cheese-able...?)

(Remark 3: the secret sharing scheme is actually hinted the solution because the way to recover the secret is CRT + LLL. I didn't want this to happen, but having only a single secret is a bit scary to me because solving one random DLP is enough to get flag...)

## Final notes

I didn't expect that VSS has more solves than LFSR, until the contestants show me that there are plenty of ways to approach VSS. Hope you enjoy the challenges and writeups. If you have any quesitons or alternative solutions, feel free to contact me!