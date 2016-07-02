---
layout: post
title: Auto-Encoding Variational Bayes [arXiv:1312.6114]
category: Chainer
tags:
- 変分ベイズ
excerpt_separator: <!--more-->
---

## 概要

- [Auto-Encoding Variational Bayes](http://arxiv.org/abs/1312.6114) を読んだ

<!--more-->

## はじめに

最近名前をよく聞くようになった変分オートエンコーダ（Variational AutoEncoder）の基礎となる**確率的勾配変分ベイズ（SGVB）**について自分なりにまとめます。

## 参考

- [20150717-suzuki.pdf](http://deeplearning.jp/wp-content/uploads/2014/04/20150717-suzuki.pdf)
	- SGVB
- [vb-nlp-tutorial.pdf](http://chasen.org/~daiti-m/paper/vb-nlp-tutorial.pdf)
	- 変分下限

## 問題設定

データを$\boldsymbol x$、隠れ変数を$\boldsymbol z$、パラメータを$\boldsymbol \theta$とし、同時確率分布$p_{\boldsymbol \theta}(\boldsymbol x, \boldsymbol z) = p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)p_{\boldsymbol \theta}(\boldsymbol z)$を推定します。

周辺尤度$p_{\boldsymbol \theta}(\boldsymbol x) = \int p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)p_{\boldsymbol \theta}(\boldsymbol z)d\boldsymbol z$が計算困難な場合、$p_{\boldsymbol \theta}(\boldsymbol x)$は$\boldsymbol \theta$で微分できないため、直接$\boldsymbol \theta$を最適化することはできません。

また事後分布$p_{\boldsymbol \theta}(\boldsymbol z\mid\boldsymbol x) = p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)p_{\boldsymbol \theta}(\boldsymbol z)/p_{\boldsymbol \theta}(\boldsymbol x)$も困難であり、EMアルゴリズムを使えません。

さらにデータが大量にあるので、時間のかかるサンプリングベースな手法は使いたくありません。

本論文ではこれらの問題を解決するための手法を提案します。

## 認識モデル

上記の問題の解決に関して、認識モデル$q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)$を導入します。

これは真の事後分布$p_{\boldsymbol \theta}(\boldsymbol z\mid\boldsymbol x)$の近似であり、パラメータ$\boldsymbol \phi$は$\boldsymbol \theta$と同時に学習させます。

## 変分下限（変分下界）

$\boldsymbol z$の対数周辺尤度は以下のように変形できます。

$$
	\begin{align}
		{\rm log}p_{\boldsymbol \theta}(\boldsymbol x) &= log\int p_{\boldsymbol \theta}(\boldsymbol x\mid \boldsymbol z)p_{\boldsymbol \theta}(\boldsymbol z)d\boldsymbol z\\
		&= log\int q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)\frac{p_{\boldsymbol \theta}(\boldsymbol x\mid \boldsymbol z)p_{\boldsymbol \theta}(\boldsymbol z)}{q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)}d\boldsymbol z\\
		&\geq\int q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x){\rm log}\frac{p_{\boldsymbol \theta}(\boldsymbol x\mid \boldsymbol z)p_{\boldsymbol \theta}(\boldsymbol z)}{q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)}d\boldsymbol z\\
		&=\int q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)
		\biggl\{
		{\rm log}\frac{p_{\boldsymbol \theta}(\boldsymbol z)}{q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)}+{\rm log}p_{\boldsymbol \theta}(\boldsymbol x\mid \boldsymbol z)
		\biggr\}d\boldsymbol z\\
		&=\int q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x){\rm log}p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)d\boldsymbol z-\int q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x){\rm log}\frac{q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)}{p_{\boldsymbol \theta}(\boldsymbol z)}d\boldsymbol z\\
		&=\double E_{\boldsymbol z \sim q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)}[{\rm log}p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)] - D_{KL}(q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)||p_{\boldsymbol \theta}(\boldsymbol z))
	\end{align}
$$

式(2)から(3)への変形にはイェンゼンの不等式を用います。

$\double E[\cdot]$は期待値、$D_{KL}$はKLダイバージェンスを表します。

式(6)は${\rm log}p_{\boldsymbol \theta}(\boldsymbol x)$の下限値を表しているため、これを増加させる$\boldsymbol \theta$を探せばよいことになります。

### 第一項について

$\double E_{\boldsymbol z \sim q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)}[{\rm log}p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)]$は、あるデータ$\boldsymbol x^{(i)}$があるときに、$q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)$からサンプリングした$\boldsymbol z^{(i)}$を用いて、$p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)$からサンプリングして得られた$\boldsymbol x'$が、もとの$\boldsymbol x^{(i)}$である度合いを表しています。

つまり、$\boldsymbol x \to \boldsymbol z \to \boldsymbol x$のようなオートエンコーダとしてうまく機能する度合いを表しているため、この項は復号誤差と呼ばれたりします。

### 第二項について

$D_{KL}(q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)\|\|p_{\boldsymbol \theta}(\boldsymbol z))$は常に0以上の値を取ります。

これは$q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)$が事前分布$p_{\boldsymbol \theta}(\boldsymbol z)$からどれだけ離れているかを表しています。つまり、この項が0になれば、$q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)$は$p_{\boldsymbol \theta}(\boldsymbol z)$に一致します。

$q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)$を、$\boldsymbol x$に無関係な$\boldsymbol z$の事前分布$p_{\boldsymbol \theta}(\boldsymbol z)$に近づけることで過学習を防ぐことができると考えられるので、この項は正則化項と呼ばれています。

### 誤差関数

式(6)は尤度なので、値が大きくなることが望ましいです。

そのため誤差関数は負の対数周辺尤度となります。

$$
	\begin{align}
		{\cal L}(\boldsymbol \theta, \boldsymbol \phi, \boldsymbol x) = -{\rm log}p_{\boldsymbol \theta}(\boldsymbol x)
	\end{align}
$$

## Stochastic Gradient Variational Bayes(SGVB)

式(7)の誤差関数を、それぞれのパラメータ$\boldsymbol \theta$、$\boldsymbol  \phi$で微分することを考えます。

### reparameterization trick

確率変数$\tilde {\boldsymbol z} \sim q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x)$を、微分可能な関数$q_{\boldsymbol \phi}(\boldsymbol \epsilon, \boldsymbol x)$を用いて以下のように表します。

$$
	\begin{align}
		\boldsymbol \epsilon &\sim p(\boldsymbol \epsilon)\nonumber\\
		\tilde {\boldsymbol z} &= q_{\boldsymbol \phi}(\boldsymbol \epsilon, \boldsymbol x)\\
	\end{align}
$$

つまり、$\boldsymbol x$を入力として取る関数に確率的なノイズ$\boldsymbol \epsilon$を乗せることで、サンプリングを決定論的に求めることができます。

たとえばある隠れ変数$\tilde z$が、データ$x$によって決まる平均$\mu(x)$、分散$\sigma(x)^2$の正規分布に従っているとします。

この時、$\tilde z$のサンプリングは以下のように行います。

$$
	\begin{align}
		\epsilon &\sim {\cal N}(0, 1)\nonumber \\
		\tilde z &= \mu(x) + \epsilon\cdot\sigma(x)\\
	\end{align}
$$

このようにすれば、$\mu(x)$や$\sigma(x)$のそれぞれのパラメータで$\tilde z$を微分することができます。

### Monte Carlo estimates

ここでは簡単のため${\rm log}p_{\boldsymbol \theta}(\boldsymbol x\mid\boldsymbol z)$を単に$f(\boldsymbol z)$と表記します。

式(6)の期待値部分は、データ$\boldsymbol x^{(i)}$に対して、reparameterization trickを用いて以下のように近似できます。

$$
	\begin{align}
		\double E_{\boldsymbol z \sim q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x^{(i)})}[f(\boldsymbol z)] &= \double E_{p(\boldsymbol \epsilon)}\bigl[f\bigl(g_{\boldsymbol \phi}(\boldsymbol \epsilon, \boldsymbol x^{(i)})\bigr)\bigr]\\
		&\simeq \frac{1}{L}\sum_{l=1}^{L}f\bigl(g_{\boldsymbol \phi}(\boldsymbol \epsilon^{(l)}, \boldsymbol x^{(i)})\bigr)\\
		{\rm where}\hspace{10pt}\boldsymbol \epsilon^{(l)} &\sim p(\boldsymbol \epsilon)\nonumber
	\end{align}
$$

これは、$L$個の$\boldsymbol z^{(l)}$（$l=0,1,...,L$）を$q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x^{(i)})$からサンプリングし、それぞれの$\boldsymbol z^{(l)}$とデータ$\boldsymbol x^{(i)}$から求めた${\rm log}p_{\boldsymbol \theta}(\boldsymbol x^{(i)}\mid\boldsymbol z^{(l)})$の平均を期待値$\double E$の推定として用いていることを表しています。

そもそも期待値は本質的には平均のことですので、この推定は当然のことです。

サイコロで例えると、$L=1$の場合、1回だけ振って期待値を推定することに相当します。

6の目が出れば期待値の推定は6となります。

$L=10000$もあれば、出た目の平均は真の期待値$3.5$に近づくでしょう。

### SGVB

上記のモンテカルロサンプリングを用いることによって、データ$\boldsymbol x^{(i)}$に対する誤差関数は最終的に以下のようになります。


$$
	\begin{align}
		\boldsymbol \epsilon^{(l)} &\sim p(\boldsymbol \epsilon)\nonumber\\
		\boldsymbol z^{(i, l)} &= g_{\boldsymbol \phi}(\boldsymbol \epsilon^{(i, l)}, \boldsymbol x^{(i)})\\
		{\cal L}(\boldsymbol \theta, \boldsymbol \phi, \boldsymbol x^{(i)}) &= -{\rm log}p_{\boldsymbol \theta}(\boldsymbol x^{(i)})\\
		&\simeq D_{KL}(q_{\boldsymbol \phi}(\boldsymbol z\mid\boldsymbol x^{(i)})||p_{\boldsymbol \theta}(\boldsymbol z)) - \frac{1}{L}\sum_{l=1}^{L}\bigl({\rm log}p_{\boldsymbol \theta}(\boldsymbol x^{(i)} \mid \boldsymbol z^{(i, l)})\bigr)\\
	\end{align}
$$

## 応用例

- [変分オートエンコーダ](/2016/07/02/semi-supervised-learning-with-deep-generative-models/)
- 変分RNN（開発中）