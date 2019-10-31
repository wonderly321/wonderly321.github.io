---
layout: page
title: About
tagline: A few more words about this theme
permalink: /about.html
ref: About
order: 0
---
% !TEX TS-program = xelatex
% !TEX encoding = UTF-8 Unicode
% !Mode:: "TeX:UTF-8"

\documentclass{resume}
\usepackage{zh_CN-Adobefonts_external} % Simplified Chinese Support using external fonts (./fonts/zh_CN-Adobe/)
%\usepackage{zh_CN-Adobefonts_internal} % Simplified Chinese Support using system fonts
\usepackage{linespacing_fix} % disable extra space before next section
\usepackage{cite}

\begin{document}
\pagenumbering{gobble} % suppress displaying page number

\name{wonderly321}

\basicInfo{
  \email{duo0702@live.cn} \textperiodcentered\ 
  \phone{(+86) 185-xxxx-1942} \textperiodcentered\ 
  \github[fieryzig]{https://www.github.com/fieryzig}}
 
\section{\faGraduationCap\  教育背景}
\datedsubsection{\textbf{中国科学院大学计算技术研究所}, 北京}{2017 -- 至今}
\textit{在读硕士研究生}\ 计算机应用技术, 预计 2020 年 6 月毕业
\datedsubsection{\textbf{武汉大学}, 湖北, 武汉}{2013 -- 2017}
\textit{学士}\ 计算机学院，年级第二名获得保研资格

\section{\faUsers\ 实习/项目经历}
\datedsubsection{\textbf{一流科技（Oneflow）有限公司实习}, 北京}{2019年1月 -- 至今}
\role{实习}{Python开发 和 C++开发}
\begin{itemize}
  \item 项目介绍：Oneflow分布式深度学习框架Python接口开发
  \item 个人职责：参与Python API的设计与实现；独立完成Tensorflow模型转换至Oneflow的工具
\end{itemize}
\datedsubsection{\textbf{Everitoken区块链项目实习}, 远程}{2018年6月 -- 2018年11月}
\role{实习}{Python开发}
\begin{itemize}
  \item 项目介绍：Everitoken区块链项目
  \item 个人职责：独立开发Python SDK；独立开发用于负载和性能测试的服务器
  \item 项目地址：https://github.com/everitoken/evt
\end{itemize}
\datedsubsection{\textbf{上海欢乐互娱有限公司实习}, 上海}{2016年11月 -- 2017年1月}
\role{实习}{游戏开发}
\begin{itemize}
  \item 项目介绍：参与2D回合游戏，棋牌游戏等手游的开发制作
  \item 个人职责：使用Unity3D和Lua，独立开发双人网络版flappybird客户端,C++开发网络服务器；参与麻将手游客户端UI的开发；参与回合制手游“疯狂坦克”客户端的开发
\end{itemize}
\datedsubsection{\textbf{人机交互课程设计} }{2018年5月}
\role{个人项目}{游戏开发}
\begin{itemize}
  \item 项目介绍：体感吃豆人游戏的开发
  \item 个人职责：通过结合dlib人脸检测技术与使用Unity3D独立开发游戏，使游戏者通过头部运动参与游戏获得趣味性
  \item 项目地址：https://github.com/fieryzig/head-pacman
\end{itemize}
\datedsubsection{\textbf{其他个人项目}}{}
\begin{itemize}
    \item 网格简化及渲染 https://github.com/fieryzig/MeshSimple
    \item 光线追踪渲染器 https://github.com/fieryzig/RTRender
\end{itemize}


% Reference Test
%\datedsubsection{\textbf{Paper Title\cite{zaharia2012resilient}}}{May. 2015}
%An xxx optimized for xxx\cite{verma2015large}
%\begin{itemize}
%  \item main contribution
%\end{itemize}

\section{\faCogs\ 个人技能}
% increase linespacing [parsep=0.5ex]
\begin{itemize}[parsep=0.5ex]
  \item C/C++ == Python > C\# > Lua  
  \item 英语四级:580 英语六级:492
  \item 熟悉Linux,Unity3D,Tensorflow,Blender
\end{itemize}

\section{\faTag\ 获奖情况}
\begin{itemize}[parsep=0.5ex]
\item{2015年国际大学生程序设计竞赛（ACM-ICPC）长春赛区 \textit{金奖}}
\item{2015年国际大学生程序设计竞赛（ACM-ICPC）上海邀请赛 \textit{金奖}}
\item{2015年国际大学生程序设计竞赛（ACM-ICPC）北京赛区 \textit{银奖}}
\item{2016年全国大学生信息安全大赛 \textit{二等奖}}
\end{itemize}

\section{\faBook\ 论文发表}
%increase linespacing [parsep=0.5ex]
\begin{itemize}[parsep=0.5ex]
  \item Replay attack detection based on distortion by loudspeaker for voice authentication (Multimedia Tools and Application 2018 第二作者)
  \item Joint Prediction for Kinematic Trajectories in Vehicle-Pedestrian-Mixed Scenes
 (ICCV2019 第二作者)
\end{itemize}

%% Reference
%\newpage
%\bibliographystyle{IEEETran}
%\bibliography{mycite}
\end{document}
