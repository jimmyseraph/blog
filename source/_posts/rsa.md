---
title: rsa
author: 六道先生
date: 2022-03-04 13:35:49
categories: 
- IT
tags:
- 加密
- RSA
+ description: 一个简单的RSA非对称性加密的java示例
---
# 一个简单的RSA非对称性加密的java示例
```Java
package com.liudao.utils;

import java.io.UnsupportedEncodingException;
import java.security.InvalidKeyException;
import java.security.Key;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.util.HashMap;
import java.util.Map;

import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;

public class AsymEncryption {
	
	public static Map<String, Key> generatorRSAKeyPair(int keysize) throws 
		NoSuchAlgorithmException {
		Map<String, Key> keyMap = new HashMap<>();
		KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
		keyPairGenerator.initialize(keysize);
		KeyPair keyPair = keyPairGenerator.generateKeyPair();
		keyMap.put("publicKey", keyPair.getPublic());
		keyMap.put("privateKey", keyPair.getPrivate());
		return keyMap;
	}
	
	public static String publicKeyRSAEncode(String content, PublicKey publicKey) throws 
		NoSuchAlgorithmException, 
		NoSuchPaddingException, 
		InvalidKeyException,
		IllegalBlockSizeException, 
		BadPaddingException, 
		UnsupportedEncodingException {
		Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
		cipher.init(Cipher.ENCRYPT_MODE, publicKey);
		byte[] bytes = cipher.doFinal(content.getBytes("utf-8"));
		return StringUtils.byteArray2HexString(bytes);
	}
	
	public static String privateKeyRSAEncode(String content, PrivateKey privateKey) throws 
		NoSuchAlgorithmException, 
		NoSuchPaddingException, 
		InvalidKeyException,
		IllegalBlockSizeException, 
		BadPaddingException, 
		UnsupportedEncodingException {
		Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
		cipher.init(Cipher.ENCRYPT_MODE, privateKey);
		byte[] bytes = cipher.doFinal(content.getBytes("utf-8"));
		return StringUtils.byteArray2HexString(bytes);
	}
	
	public static String publicKeyRSADecode(String content, PublicKey publicKey) throws 
		NoSuchAlgorithmException, 
		NoSuchPaddingException, 
		InvalidKeyException,
		IllegalBlockSizeException, 
		BadPaddingException, 
		UnsupportedEncodingException {
		Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
		cipher.init(Cipher.DECRYPT_MODE, publicKey);
		byte[] bytes = cipher.doFinal(StringUtils.hexString2ByteArray(content));
		return new String(bytes,"utf-8");
	}

	public static String privateKeyRSADecode(String content, PrivateKey privateKey) throws 
		NoSuchAlgorithmException, 
		NoSuchPaddingException, 
		InvalidKeyException,
		IllegalBlockSizeException, 
		BadPaddingException, 
		UnsupportedEncodingException {
		Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
		cipher.init(Cipher.DECRYPT_MODE, privateKey);
		byte[] bytes = cipher.doFinal(StringUtils.hexString2ByteArray(content));
		return new String(bytes,"utf-8");
	}
	
	public static void main(String[] args) throws Exception{
		Map<String, Key> keyMap = generatorRSAKeyPair(1024);
		PublicKey publicKey = (PublicKey)keyMap.get("publicKey");
		PrivateKey privateKey = (PrivateKey)keyMap.get("privateKey");
		System.out.println("publicKey -> "+publicKey);
		System.out.println("privateKey -> "+privateKey);
		String dest = "123456";
		String s1 = publicKeyRSAEncode(dest, publicKey);
		System.out.println("Encrypted with PublicKey -> " + s1);
		String s2 = privateKeyRSADecode(s1, privateKey);
		System.out.println("Decrypted with privateKey -> " + s2);
		String s3 = privateKeyRSAEncode(dest, privateKey);
		System.out.println("Encrypted with PrivateKey -> " + s3);
		String s4 = publicKeyRSADecode(s3, publicKey);
		System.out.println("Decrypted with publicKey -> " + s4);
	}
}
```