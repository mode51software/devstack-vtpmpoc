# Devstack vTPM Client Certificate Injection PoC

This proof of concept demonstrates how a hypervisor or orchestrator can inject an AK (Attestation Key)
and a signed client certificate into a vTPM (virtual TPM) that can subsequently be used by a VM 
to issue new certificates.

The key goal is to remove hard coded credentials from VMs that may be needed to access CA issuers.

Devstack Wallaby is currently supported. The instructions will be updated to support Devstack Xena.

[Step by step instructions are available here](SETUP.md).